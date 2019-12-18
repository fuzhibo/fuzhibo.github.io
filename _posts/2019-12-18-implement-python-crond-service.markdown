---
layout: post
title:  "实现一个 python 版本的 Crond Service"
date:   2019-12-18 12:18:29 +0800
categories: python,crond
---
&emsp;&emsp;在微服务的实践中，所有的应用都被打包成了容器。但是总会遇到一些作业定时调度的需求，在不知道有 `APScheduler` 的情况下通常的办法都会想到用 Linux 系统中自带的 Crond Service 来完成相应的作业调度功能。那么如果要将这部分的实现迁移到容器中来，大多数情况下会选择同时集成 Crond Service 到容器。这就破坏了`一个进程一个容器`的容器设计思想，并且在实际的运行过程中 Crond Service 的运行也不是足够的稳定。默认情况下 Crond Service 启动后，会自动在 `/var/run/` 中创建一个 `crond.pid` 文件。如果为镜像新增了什么配置并且使用 `docker commit` 重新打包了之后，这个 `crond.pid` 文件也会被打包并会干扰到下次的容器启动或者 `docker exec` 的操作。  
&emsp;  
&emsp;&emsp;在知道了 `APScheduler` 后，一个办法是通过重构将原有的调度全部放到代码中，但是这种重构带来的问题是未来每次新增一个作业调度都要增加代码。无疑是大大的增加了服务开发的时间。另外一个办法是考虑实现一个 python 版本的 Crond Service 服务。  
&emsp;  
&emsp;&emsp;因为存在 `python-crontab` 这样的库，使用 python 管理 `crontab` 文件的复杂度被大大的降低，而同时结合 `APScheduler` 就可以实现一个 python 版本的 Crond Service 服务。  
```python
import os
import datetime
import json
import time
import requests
import traceback
import hashlib
from watchdog.observers import Observer
from crontab import CronTab
from watchdog.events import FileSystemEventHandler
from threading import Thread, Event
import queue as Queue
from apscheduler.schedulers.background import BackgroundScheduler
from cmop.common.log import info, warn, debug, error
from cmop.common.rabbitmq_interface import RabbitmqService
from cmop.common.utils import RedisClient, run_local_command, byteify

REDIS_HOST = "redis-master"
REDIS_PORT = 6379
SYSTEM_CRONTAB = "/etc/crontab"
CROND_SERVICE_PREFIX = "crond-service-cronjob-cache"

MSGQUEUE = Queue.Queue()
CROND_RESTART_EVENT = Event()


class SchedulerManagement(object):

    scheduler_dict = {}
    start_flag_dict = {}

    def __init__(self, key_prefix):
        self._scheduler_key_prefix = key_prefix
        if not self.scheduler:
            info(f"Init scheduler: {self._scheduler_key_prefix}")
            self.start_flag = False
            self.scheduler = BackgroundScheduler()
            self.scheduler.configure(
                job_defaults={
                    "coalesce": True,
                    "max_instances": 3000,
                    "misfire_grace_time": 60,
                }
            )
            self.scheduler.add_jobstore(
                "redis",
                jobs_key="{0}.jobs".format(key_prefix),
                run_times_key="{0}.run_times".format(key_prefix),
                host="redis-master",
                port=6379,
            )
        else:
            info(f"Scheduler: {self._scheduler_key_prefix} has already started, ignore initialize...")

    @property
    def scheduler(self):
        if self._scheduler_key_prefix not in SchedulerManagement.scheduler_dict:
            return None
        else:
            return SchedulerManagement.scheduler_dict[self._scheduler_key_prefix]

    @scheduler.setter
    def scheduler(self, value):
        if not value:
            # delete related instance
            info(f"Scheduler {self._scheduler_key_prefix} will be destroy...")
            del SchedulerManagement.scheduler_dict[self._scheduler_key_prefix]
        SchedulerManagement.scheduler_dict[self._scheduler_key_prefix] = value

    @property
    def start_flag(self):
        if self._scheduler_key_prefix not in SchedulerManagement.start_flag_dict:
            SchedulerManagement.start_flag_dict[self._scheduler_key_prefix] = False

        return SchedulerManagement.start_flag_dict[self._scheduler_key_prefix]

    @start_flag.setter
    def start_flag(self, value):
        SchedulerManagement.start_flag_dict[self._scheduler_key_prefix] = value

    def run(self):
        if not self.start_flag:
            self.scheduler.start()
            self.start_flag = True
        info("Start scheduler [%s]..." % (self.start_flag))

    def shutdown(self):
        if self.start_flag:
            self.scheduler.shutdown()
            self.start_flag = False
            self.scheduler = None
        info("Stop scheduler...")

    def _send_schedule_notify(self, exchange, routing_key, record):
        info("Begin to send scheduled notify...")
        try:
            rs = RabbitmqService()
            rs.publish(exchange, routing_key, message=record)
        except Exception as ex:
            error(traceback.format_exc())
            error("Failed to send data %r due to %r" % (record, ex))

    def add_common_notify_callback(
        self, m="*", h="*", dom="*", mon="*", dow="*", jobid=None, cb=None, cb_args=None
    ):
        job = self.scheduler.get_job(jobid)
        if job:
            info(f"The job {jobid} has already exists, ignore...")
        else:
            info("Add notify callback...")
            self.scheduler.add_job(
                cb,
                args=cb_args,
                trigger="cron",
                minute=m,
                hour=h,
                day=dom,
                month=mon,
                day_of_week=dow,
                start_date=datetime.datetime.now(),
                id=jobid,
            )
            self.scheduler.resume()

    def add_common_notify_callback_once(
        self, run_date, jobid=None, cb=None, cb_args=None
    ):
        info("Add once notify callback...")
        self.scheduler.add_job(
            cb, args=cb_args, trigger="date", run_date=run_date, id=jobid
        )
        self.scheduler.resume()

    def remove_all_jobs(self):
        self.scheduler.remove_all_jobs()

    def remove_job(self, jobid=None):
        if jobid:
            job = self.scheduler.get_job(jobid)
            if job:
                self.scheduler.remove_job(jobid)
        else:
            self.scheduler.remove_all_jobs()

    def add_schedule(
        self, m="*", h="*", dom="*", mon="*", dow="*", jobid=None, cb=None, cb_args=()
    ):
        info("Add new scheduled job...")
        cb = cb or self._send_schedule_notify
        if jobid:
            redis_clt = RedisClient(REDIS_HOST, REDIS_PORT, decode_responses=True)
            body = redis_clt.client.hget(self._scheduler_key_prefix, jobid)
            if body:
                cb_args += (body,)
        self.add_common_notify_callback(
            m=m, h=h, dom=dom, mon=mon, dow=dow, jobid=jobid, cb=cb, cb_args=cb_args
        )

    def add_schedule_once(self, run_date=None, jobid=None, cb=None, cb_args=()):
        info("Add new once scheduled job...")
        cb = cb or self._send_schedule_notify
        if jobid:
            redis_clt = RedisClient(REDIS_HOST, REDIS_PORT, decode_responses=True)
            body = redis_clt.client.hget(self._scheduler_key_prefix, jobid)
            if body:
                cb_args += (body,)
        if run_date:
            self.add_common_notify_callback_once(
                run_date=run_date, jobid=jobid, cb=cb, cb_args=cb_args
            )
        else:
            info("Failed to get schedule run date, ignore cron config...")


class CronTabEventHandler(FileSystemEventHandler):
    """Logs all the events captured."""

    def on_moved(self, event):
        super(CronTabEventHandler, self).on_moved(event)

        what = "directory" if event.is_directory else "file"
        debug(f"Moved {what}: from {event.src_path} to {event.dest_path}")

    def on_created(self, event):
        super(CronTabEventHandler, self).on_created(event)

        what = "directory" if event.is_directory else "file"
        debug(f"Created {what}: {event.src_path}")

    def on_deleted(self, event):
        super(CronTabEventHandler, self).on_deleted(event)

        what = "directory" if event.is_directory else "file"
        debug(f"Deleted {what}: {event.src_path}")

    def on_modified(self, event):
        super(CronTabEventHandler, self).on_modified(event)
        redis_clt = RedisClient(REDIS_HOST, REDIS_PORT, decode_responses=True)
        if not event.is_directory and event.src_path == SYSTEM_CRONTAB:
            debug(f"Modified file: {event.src_path}")
            # read the crontab file and set it to apscheduler
            # clean redis
            redis_clt.client.delete(CROND_SERVICE_PREFIX)
            system_cron = CronTab(tabfile=SYSTEM_CRONTAB, user=False)
            cron_jobs = []
            for job in system_cron:
                job_mark = f"{str(job.minute)},{str(job.hour)},{str(job.dom)},{str(job.month)},{str(job.dow)},{job.user},{job.command}"
                jobid = f"crondServiceTask-{hashlib.md5(job_mark.encode('utf-8')).hexdigest()[8:-8]}"
                MSGQUEUE.put(
                    (
                        str(job.minute),
                        str(job.hour),
                        str(job.dom),
                        str(job.month),
                        str(job.dow),
                        jobid,
                        job.command,
                    )
                )
                cron_jobs.append(
                    [
                        str(job.minute),
                        str(job.hour),
                        str(job.dom),
                        str(job.month),
                        str(job.dow),
                        job.user,
                        job.command,
                    ]
                )
            else:
                # record in redis
                redis_clt.client.set(CROND_SERVICE_PREFIX, json.dumps(cron_jobs))
                # clean current jobs and restart crond service
                CROND_RESTART_EVENT.set()


class CrondService(Thread):
    """Start a thread run like crond service"""

    def __init__(self, name, **kwargs):

        super(CrondService, self).__init__(name=name)

    def _recovery_system_crontab(self):
        # check the file has already existed
        if not os.path.exists(SYSTEM_CRONTAB):
            info(f"There is no related file {SYSTEM_CRONTAB}, will create one...")
            fp = open(SYSTEM_CRONTAB, "x")
            fp.close()
        # recovery from redis and write to /etc/crontab
        info("Recovery crond setting from redis...")
        redis_clt = RedisClient(REDIS_HOST, REDIS_PORT, decode_responses=True)
        body = redis_clt.client.get(CROND_SERVICE_PREFIX)
        crond_info = json.loads(body, object_hook=byteify) if body else None
        if crond_info:
            system_cron = CronTab(tabfile=SYSTEM_CRONTAB, user=False)
            for cron in crond_info:
                m, h, dom, mon, dow, user, cmd = cron
                job_mark = (
                    f"{str(m)},{str(h)},{str(dom)},{str(mon)},{str(dow)},{user},{cmd}"
                )
                jobid = f"crondServiceTask-{hashlib.md5(job_mark.encode('utf-8')).hexdigest()[8:-8]}"
                newjob = system_cron.new(
                    command=cmd, comment=f"crond service {jobid}", user=user
                )
                newjob.setall(f"{str(m)} {str(h)} {str(dom)} {str(mon)} {str(dow)}")
                MSGQUEUE.put(
                    (
                        str(m),
                        str(h),
                        str(dom),
                        str(mon),
                        str(dow),
                        jobid,
                        cmd,
                    )
                )
            else:
                system_cron.write()
                info("Recovery crond setting completed.")
        else:
            info("There is no related cron job record in cache, ignore recovery...")

    def run(self):
        info("Begin to start crond service monitor...")
        crond_tid = None
        # recovery the cron job
        self._recovery_system_crontab()
        # create a new thread to run crond service
        while True:
            if not crond_tid:
                crond_tid = Thread(target=self._crond_service_run)
                crond_tid.start()
                # try some counts to wait the thread alive
                for _ in range(3):
                    if crond_tid.isAlive():
                        break
                    else:
                        warn("crond service thread is not ready, waitting..")
                        time.sleep(2)
                else:
                    warn("crond service can not be ready.")
                    break
                info("Crond service started...")
            else:
                # monitor the event and restart the thread
                CROND_RESTART_EVENT.wait()
                # restart the crond service
                # waith the thread stop
                info("Waitting for Crond service stop...")
                crond_tid.join()
                crond_tid = None
                info("Crond service stopped...")
                CROND_RESTART_EVENT.clear()
        else:
            info("Crond service has quit...")

    def _crond_service_run(self):

        info("Begin to start Crond Service...")
        path = "/etc"
        event_handler = CronTabEventHandler()
        observer = Observer()
        info("Begin to watch /etc/crontab file...")
        observer.schedule(event_handler, path, recursive=True)
        observer.start()
        key_prefix = "crond_service_schedule"
        sch_manage = SchedulerManagement(key_prefix=key_prefix)
        sch_manage.run()
        # remove all jobs
        sch_manage.remove_job()
        while True:
            msg_consumed = False
            try:
                # get the crontab setting and set to
                # apscheduler
                if not MSGQUEUE.empty():
                    m, h, dom, mon, dow, jobid, cmd = MSGQUEUE.get(
                        block=False, timeout=3
                    )
                    msg_consumed = True
                    sch_manage.add_schedule(
                        m, h, dom, mon, dow, jobid, run_local_command, (cmd,)
                    )
                else:
                    # check the Event and decide if quit
                    if CROND_RESTART_EVENT.wait(timeout=1):
                        info("Thread recive the quit event, will quite...")
                        break
            except Exception as ex:
                error(traceback.format_exc())
                error(f"Failed to run crond service scheduler manager due to {str(ex)}")
                break
            finally:
                if msg_consumed:
                    MSGQUEUE.task_done()

        observer.stop()
        observer.join()
        sch_manage.shutdown()

if __name__ == '__main__':
    # start crond service
    crond = CrondService("crond-service")
    crond.start()
```
&emsp;&emsp;简单的描述一下这个小模块的工作流程：  
{% mermaid %}
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
{% endmermaid %}
<script>mermaid.initialize({startOnLoad:true});</script>
