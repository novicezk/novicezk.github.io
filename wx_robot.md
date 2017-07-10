# wx-robot
使用itchat让自己的微信号使用机器人自动回复，支持语音识别，但因为itchat接口基于微信网页客户端，没有发送语音的功能（发送mp3格式的语音会显示成文件），所以只能识别语音然后回复文本。

## 运行环境（ubuntu）：
1. python3.5
2. pip3： sudo apt-get install python3-pip
3. itchat： pip3 install itchat
4. ffmpeg： sudo apt-get install ffmpeg
5. 在当前用户主目录下创建tmp文件夹

## 启动：
1. python3.5 wx.py
2. 微信扫描终端的二维码登录
3. 私聊自己发送stop可暂停程序，发送run再次运行
4. 私聊、群聊有人@自己都会触发机器人回复

## 注意事项：
1. 若出现二维码缺失，请尝试修改itchat.auto_login(enableCmdQR=2, hotReload=True)为itchat.auto_login(enableCmdQR=True, hotReload=True),若系统为图形化操作系统，可改为itchat.auto_login(hotReload=True)
2. 登录网页版微信，程序会终止
3. tl_api_key和baidu_api_key为测试使用，建议[注册图灵机器人](http://www.tuling123.com/register/index.jhtml)，[注册百度语音](http://yuyin.baidu.com/)或[讯飞语音](http://www.xfyun.cn/)，使用自己的apiKey

## wx.py
```
import itchat
import requests
import os
import base64
import subprocess
import json
from itchat.content import *

HOME = os.environ['HOME']
is_run = True

tl_api_key = '107adc2db7f34eaaa3a552fe359b2d6a'
tl_api_url = 'http://www.tuling123.com/openapi/api'

baidu_api_key = "ka0XBT54ZgyX6OrBKbR09228"
baidu_secret_key = "1e2119959703c66afcadc2ff1f9b4dc0"
baidu_speech_recognition_url = "http://vop.baidu.com/server_api"

req = requests.get(url='https://openapi.baidu.com/oauth/2.0/token',
                   params={'grant_type': 'client_credentials', 'client_id': baidu_api_key,
                           'client_secret': baidu_secret_key})
access_token = req.json()['access_token']


def chat_robot(msg, user_id):
    data = {'key': tl_api_key, 'info': msg, 'userid': user_id}
    r = requests.post(tl_api_url, data=json.dumps(data))
    return r.json()['text']


def speech_recognition(file_name, user_id):
    f = open(file_name, 'rb')
    speech = base64.b64encode(f.read()).decode('utf-8')
    f.close()
    size = os.path.getsize(file_name)
    data = {'format': 'wav', 'rate': 16000, 'channel': '1', 'token': access_token, 'cuid': user_id, 'len': size,
            'speech': speech}
    r = requests.post(baidu_speech_recognition_url, data=json.dumps(data))
    return r.json()


@itchat.msg_register(TEXT)
def private_chat(msg):
    global is_run
    if (itchat.web_init()['User']['UserName'] == msg['FromUserName']):
        if (msg['Text'] == 'stop'):
            is_run = False
            print('stop')
        elif (msg['Text'] == 'run'):
            is_run = True
            print('run')
    elif (is_run):
        robot_answer = chat_robot(msg['Text'], msg['FromUserName'][1:33])
        itchat.send(robot_answer, msg['FromUserName'])


@itchat.msg_register(RECORDING)
def record_chat(msg):
    global is_run
    if (is_run):
        file_name = HOME + '/tmp/' + msg['FileName']
        msg['Text'](file_name)
        new_file_name = file_name.replace('.mp3', '.wav')
        subprocess.call('ffmpeg -i ' + file_name + ' -acodec pcm_s16le -ac 1 -ar 16000 ' + new_file_name, shell=True)
        user_id = msg['FromUserName'][1:33]
        result = speech_recognition(new_file_name, user_id)
        err_answer = '你说什么呀，我没听清[疑问]'
        if (result['err_no'] == 0):
            text = result['result'][0]
            if (text == '，'):
                itchat.send(err_answer, msg['FromUserName'])
            else:
                robot_answer = chat_robot(text, user_id)
                itchat.send(robot_answer, msg['FromUserName'])
        else:
            itchat.send(err_answer, msg['FromUserName'])

            
@itchat.msg_register(TEXT, isGroupChat=True)
def group_chat(msg):
    global is_run
    if (is_run and msg['isAt']):
        actual_nick_name = msg['ActualNickName']
        text = msg['Text'].replace('@' + actual_nick_name + '\u2005', '')
        robot_answer = '@' + actual_nick_name + ' ' + chat_robot(text, msg['ActualUserName'][1:33])
        itchat.send(robot_answer, msg['User']['UserName'])


itchat.auto_login(enableCmdQR=2, hotReload=True)
itchat.run()
itchat.dump_login_status()


```
