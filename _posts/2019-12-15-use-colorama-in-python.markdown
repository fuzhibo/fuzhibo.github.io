---
layout: post
title:  "使用 colorama 让 python 日志输出带有各种颜色标识"
date:   2019-12-15 12:18:29 +0800
categories: python,colorama
---
&emsp;&emsp;在日常的程序调试工作中，最常用的手段就是看日志。但是大量的日志输出总是让人头昏眼花，于是就有了给日志标识颜色来区分不同输出的需求。  
&emsp;  
&emsp;&emsp;在 python 中，可以使用 colorama 来完成对输出日志的颜色标注。从原来上来说本质是集成了控制台的颜色控制符，使其在 python 环境中更加的易用。不过日常使用中，多是和 logging 模块集成来方便日志输出的颜色标识。  
&emsp;  
&emsp;&emsp;在 colorama 中，主要有如下的三个模块：
|模块|参数|
| :-----: | :----- |
| Fore | BLACK, RED, GREEN, YELLOW, BLUE, MAGENTA, CYAN, WHITE, RESET |
| Back | BLACK, RED, GREEN, YELLOW, BLUE, MAGENTA, CYAN, WHITE, RESET |
| Style | DIM, NORMAL, BRIGHT, RESET_ALL |
&emsp;&emsp;其中，Fore 主要是用于设置字体本身的颜色，Back 用于设置字符的背景颜色，Style 用于设置字体本身的风格。需要注意的是，在指定的范围内设置颜色一定要在结尾加上 RESET 的操作。
```python
import re
import sys
import logging
from logging import StreamHandler
from colorama import init as colorama_init
from colorama import Fore, Back, Style


COLORS = {
    "WARNING": Fore.YELLOW,
    "INFO": Fore.WHITE,
    "DEBUG": Fore.BLUE,
    "CRITICAL": Fore.RED + Style.BRIGHT,
    "ERROR": Fore.RED,
}

COLORS_END = {
    "WARNING": Fore.RESET,
    "INFO": Fore.RESET,
    "DEBUG": Fore.RESET,
    "CRITICAL": Fore.RESET + Style.RESET_ALL,
    "ERROR": Fore.RESET,
}


class ColoredFormatter(logging.Formatter):
    def __init__(self, fmt=None, datefmt=None, style="%", use_color=True):
        super(ColoredFormatter, self).__init__(fmt, datefmt)
        self.use_color = use_color
        self.p = re.compile(r"([\"|\s])([\w]+-[\w|\d]+)([\"|\s])")
        self.http_p = re.compile(r"http[s]*://[^ |$]+")

    def format(self, record):
        levelname = record.levelname
        if self.use_color and levelname in COLORS:
            record.filename = (
                Fore.CYAN
                + Style.BRIGHT
                + record.filename
                + Fore.RESET
                + Style.RESET_ALL
            )
            record.levelname = COLORS[levelname] + levelname + COLORS_END[levelname]
            record.msg = self.p.sub(
                lambda e: e.group(1)
                + COLORS_END[levelname]
                + Fore.GREEN
                + Style.BRIGHT
                + e.group(2)
                + Fore.RESET
                + Style.RESET_ALL
                + COLORS[levelname]
                + e.group(3),
                record.msg,
            )
            if "http" in record.msg:
                record.msg = self.http_p.sub(
                    lambda e: COLORS_END[levelname]
                    + Fore.CYAN
                    + Style.BRIGHT
                    + e.group(0)
                    + Fore.RESET
                    + Style.RESET_ALL
                    + COLORS[levelname],
                    record.msg,
                )
            record.msg = COLORS[levelname] + record.msg + COLORS_END[levelname]
        return logging.Formatter.format(self, record)

def debug(msg):
    logger.debug(msg)

if __name__ == "__main__":
    colorama_init()
    handler = StreamHandler()
    handler.setLevel(logging.DEBUG)
    fmt_str = "%(asctime)s - [pid:%(process)d][tid:%(thread)d](%(filename)s:%(lineno)d) - %(levelname)s: %(message)s"
    formatter = ColoredFormatter(fmt_str)
    handler.setFormatter(formatter)
    logger = logging.getLogger(__name__)
    logger.setLevel(logging.DEBUG)
    logger.addHandler(handler)
    debug("debug")
    logger.info("Check if instance-iXt3Lce1 can be connected...")
    logger.warn(
        "warn  http://www.bing.com"
    )
    logger.error(
        '{"taskCode": "task-a912a3ef70034366a7f6dd77a9c9d4e4", "instanceId": "instance-i0dTif33"}'
    )
    logger.critical("critical")
```
&emsp;&emsp;上面这段代码，基本就是通过定制 logging 的 formatter 来花式的实现各种日志的颜色标注。  
