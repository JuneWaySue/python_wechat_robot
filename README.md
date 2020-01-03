# python_wechat_robot
python实现微信自动回复机器人+查看撤回消息

# python实现微信自动回复机器人+查看别人撤回的消息（部署到云服务器）
# 前言

 - 首先你的微信号能够登录网页版微信，才能打造你的专属个人微信号机器人，[点击跳转网页版微信登录页面](https://wx.qq.com/)
 - 类似的文章网上也都有，其实我也是受到别的文章的一些启发，因为不是每个人都想实现同样的功能的，直接套用别人的代码不严谨而且bug太多，于是就想自己动手从零开始实现一个属于自己的微信机器人，不过呢，也大同小异吧。
 - 算下来前前后后加上写这篇博客花了大概一周的时间，因为都是用零零散散的时间进行开发以及测试然后修改bug再加功能再开发，这么一个循环，从一开始的只能回复消息、到现在能够：回复特定群聊消息、特殊群聊特殊处理、回复表情包、查看所有别人撤回的消息以及操控微信机器人等等等等。


*好的，废话不多说，接下来就开始吧。*
### 一、准备
 1. `python3.7`（重中之重，后面会解释）
 2. `itchat`（直接用`pip`命令安装即可）
 3. `jupyter notebook`（随意，用你最喜欢的编译器即可，不过最后还是要把代码放在一个py文件里）
 4. `NLP`实现一个聊天机器人（限于本人没学过自然语言处理，并且空闲时间也不多，其实就是因为太难了。。那就只能先调用别人的接口啦）
### 二、开始
*ps：详情请看代码注释，若不想分函数来看也可以直接看完整代码*
- **定义获取好友的昵称和好友的备注函数**

```python
def get_friendname():
    friends_name={} #存储好友的微信昵称和备注
    friends=itchat.get_friends(update=True) #返回的是一个存储所有好友信息的列表，每个好友的信息都是用一个字典来存放
    for friend in friends[1:]: #第一个为自己，所以这里排除了自己
        friends_name.update({friend['UserName']:{'nickname':friend['NickName'],'remarkname':friend['RemarkName']}})
    return friends_name
```
- **定义群聊信息的函数**
*ps：这个获取群聊信息的函数只能读取到你保存到通讯录中的群聊，那些没有保存到通讯录中的是显示不出来的，不过不影响获取群聊信息，它只是没有显示而已，后面添加特定群聊就算是没有保存通讯录的都是可以添加的，一样可以回复特定群聊。*
```python
def get_username():
    chatrooms=itchat.get_chatrooms(update=True) #返回的是一个所有群聊的信息的列表，每个群聊信息都是用一个字典来存放
    user_name=[] #接收特定群聊@本人的消息，并回复；存放特定群聊的username
    all_user_name=[] #存放全部群聊的username
    vip=[] #存放特定群聊的名称
    if os.path.exists('./vip.txt'):
        with open('./vip.txt','r',encoding='utf-8') as f:
            for i in f.read().split('\n')[:-1]:
                vip.append(i)
    for chatroom in chatrooms:
        all_user_name.append(chatroom['UserName'])
        if chatroom['NickName'] in vip:
            user_name.append(chatroom['UserName'])
    return all_user_name,user_name,vip
```
- **定义获取聊天机器人返回信息的函数**[点击跳转在线聊天机器人](http://i.xiaoi.com)

```python
def get_response(msg):
    url=''#看到请求url好像涉及到一些sessionid、userid等信息，可能直接复制会用不了什么的，所以你们直接去分析一下网页即可拿到啦，把content参数format成msg即可
    headers={'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.79 Safari/537.36'}
    r=requests.get(url,headers=headers)
    response=re.findall('"body":{"fontStyle":0,"fontColor":0,"content":"(.*?)","emoticons":{}}}',r.text)[1].replace('\\r\\n','')
    return response
```
- **定义获取聊天机器人词穷时要回复消息的函数**

```python
def get_words():
    words=[]
    if os.path.exists('./words.txt'):
        with open('./words.txt','r',encoding='utf-8') as f:
            for i in f.read().split('\n')[:-1]:
                words.append(i)
    return words
```
- **定义注册消息函数（重头戏）**

```python
# 包括文本(表情符号)、位置、名片、通知、分享、图片(表情包)、语音、文件、视频
@itchat.msg_register([TEXT,MAP,CARD,SHARING,PICTURE,RECORDING,ATTACHMENT,VIDEO],isFriendChat=True,isGroupChat=True) #监听个人消息和群聊消息
def download_reply_msg(msg):
    global flag,sj,isrun,use_info #flag判断要不要进入斗图模式，sj控制斗图的时间长短，isrun判断是否启动自动回复机器人(默认运行中)，通过向传输助手发指令来控制，use_info说明文档
    all_user_name,user_name,vip=get_username()#每次接受消息时要拿到当前规定的群聊和特定群聊信息，后面用来分别做处理
    words=get_words() #拿到当前自定义回复消息的信息
    now_time=int(time.time()) #记录获取这条消息的时间，后面处理撤回消息的时候用到
    b=[] #用来记录已经过了可以撤回的时间的消息
    if len(msg_dict) != 0:
        for key,value in msg_dict.items():
            if (now_time - value['time']) >= 125: #经过验证发现消息2分钟之内才能撤回，这里为了保险起见加多5秒钟
                b.append(key)
        for eachkey in list(msg_dict.keys()):
            if eachkey in b: #要是过了撤回时间的消息是文件类型的就把它们删除，避免增加不必要的磁盘空间，盘大的请随意
                if 'file' in msg_dict[eachkey].keys():
                    os.remove(msg_dict[eachkey]['file'])
                msg_dict.pop(eachkey)
#---------------------------------------------------------
#下面开始存储各类消息，主要是用来查看别人撤回的消息，后面会用到
    if msg['Type'] in [MAP,SHARING]: #地图或者分享
        old_id=msg['MsgId']
        link=msg['Url']
        msg_dict.update({old_id:{'type':msg['Type'],'data':link,'time':now_time}})
    elif msg['Type'] in [PICTURE,RECORDING,ATTACHMENT,VIDEO]:
        if msg['ToUserName'] != 'filehelper': # 避免给文件传输助手发文件也传入字典，没必要而且传入字典只是为了防止撤回，况且它是没有撤回的
            old_id=msg['MsgId']
            file='./保存的文件/'+ msg['MsgId'] + '.' + msg['FileName'].split('.')[-1]
            msg['Text'](file)
            msg_dict.update({old_id:{'type':msg['Type'],'file':file,'time':now_time}})
        else:
            file='./保存的文件/'+ msg['FileName']
            msg['Text'](file)
    elif msg['Type'] == CARD: #名片
        old_id=msg['MsgId']
        link=re.findall('bigheadimgurl="(.*)" smallheadimgurl',str(msg))[0]
        msg_content = '来自' + msg['RecommendInfo']['Province'] +msg['RecommendInfo']['City'] + '的'+ msg['RecommendInfo']['NickName'] + '的名片'    #内容就是推荐人的昵称和性别
        if msg['RecommendInfo']['Sex'] == 1:
            msg_content += '，男的'
        else:
            msg_content += '，女的'
        msg_dict.update({old_id:{'type':msg['Type'],'head':link,'data':msg_content,'time':now_time}})
    elif msg['Type'] == TEXT: #文本
        old_id=msg['MsgId']
        text=msg['Text']
        msg_dict.update({old_id:{'type':msg['Type'],'data':text,'time':now_time}})
#---------------------------------------------------------
#下面是自动回复消息的（一切回复逻辑都在这里）
    if msg['ToUserName'] != 'filehelper': # 避免给文件传输助手发消息也自动回复
        if isrun == '运行中......': #操控机器人的，想停就停，想启动就启动，不用关掉程序，而且不影响查看撤回消息的功能
            if msg['FromUserName'] in all_user_name:
                if msg['FromUserName'] in user_name: #当消息来自特定群聊时，下面代码才会执行
                    if sj is not None:
                        if int(time.time()) - sj >= 900: #斗图时间：15分钟
                            flag=0
                            sj=None
                    if (msg['isAt'] is True)&(msg['Type'] == TEXT):
                        myname='@' + re.findall("'Self'.*?'DisplayName': '(.*?)', 'KeyWord'",str(msg))[0] if re.findall("'Self'.*?'DisplayName': '(.*?)', 'KeyWord'",str(msg))[0] != '' else '这里填你自己的微信昵称'
                        if '帅哥来斗图' in msg['Text']:
                            flag=1
                            sj=int(time.time())
                            num=random.choice(os.listdir('./表情包'))
                            msg.user.send('@img@./表情包/{}'.format(num))
                            return None
                        reply = get_response(msg['Text'].replace(myname,''))
                        if 'I am' in reply:
                            reply=reply.replace('小i机器人','your father')
                        if '小i' in reply: #这个我是不想让别人知道是小i机器人，才把它换掉的，你想换成什么随你，不换也行，就把这段代码删除即可
                            reply=reply.replace('小i','你爸爸')
                        if '机器人' in reply:
                            reply=reply.replace('机器人','')
                        if '输入' in reply:
                            if flag == 0:
                                reply='有种来斗图，输入“帅哥来斗图”即可。'
                            else:
                                reply=random.choice(words)
                        itchat.send('@%s\u2005%s' % (msg['ActualNickName'], reply), msg['FromUserName'])
                    if (msg['Type'] == PICTURE)&(flag == 1):
                        num=random.choice(os.listdir('./表情包'))
                        msg.user.send('@img@./表情包/{}'.format(num))
                else: #这里是当消息来自不是特定群聊时，要执行的代码
                    if msg['Type'] == TEXT:
                        if '收到请回复' in msg['Text']:
                            return '收到'
            else: #下面是处理个人消息的
            	#经过测试发现如果自己手动发消息到新建的群中，也会触发下面自动回复的代码，于是就要排除这个bug用下面第一个if语句
            	if msg['FromUserName'] == itchat.search_friends(nickName='这里填你自己的微信昵称')[0]['UserName']:
                    return None
                if msg['Type'] == TEXT: #下面跟处理群聊的时候差不多，就不重复了嘻嘻
                    reply = get_response(msg['Text'])
                    if 'I am' in reply:
                        reply=reply.replace('小i机器人','your father')
                    if '小i' in reply:
                        reply=reply.replace('小i','你爸爸')
                    if '机器人' in reply:
                        reply=reply.replace('机器人','')
                    if '输入' in reply:
                        reply=random.choice(words)
                    msg.user.send(reply+'\n                                    [不是本人]') # 36个空格
                elif msg['Type'] == PICTURE: #表情包回复
                    num=random.choice(os.listdir('./表情包'))
                    msg.user.send('@img@./表情包/{}'.format(num))
                elif msg['Type'] == RECORDING:
                    msg.user.send('请打字和我交流，谢谢。'+'\n                                    [不是本人]')
#---------------------------------------------------------
#下面是用来控制机器人的（给文件传输助手发指令）代码很简单，也很清晰
    else:
        if msg['Type'] == TEXT:
            if '添加vip' in msg['Text']:
                with open('./vip.txt','a',encoding='utf-8') as f:
                    f.write(msg['Text'][5:])
                    f.write('\n')
            if '查看vip' in msg['Text']:
                now_vip='\n'.join(vip)
                itchat.send('当前的vip群有：\n{0}'.format(now_vip), toUserName='filehelper')
            if '删除vip' in msg['Text']:
                if os.path.exists('./vip.txt'):
                    with open('./vip.txt','r',encoding='utf-8') as f1:
                        lines=f1.readlines()
                        with open('./vip.txt','w',encoding='utf-8') as f2:
                            for line in lines:
                                if msg['Text'][5:] != line.strip():
                                    f2.write(line)
            if '清空vip' in msg['Text']:
                with open('./vip.txt','w',encoding='utf-8') as f:
                    f.flush()
                    
            if '添加words' in msg['Text']:
                with open('./words.txt','a',encoding='utf-8') as f:
                    f.write(msg['Text'][7:])
                    f.write('\n')
            if '查看words' in msg['Text']:
                now_words='\n'.join(words)
                itchat.send('当前的words有：\n{0}'.format(now_words), toUserName='filehelper')
            if '删除words' in msg['Text']:
                if os.path.exists('./words.txt'):
                    with open('./words.txt','r',encoding='utf-8') as f1:
                        lines=f1.readlines()
                        with open('./words.txt','w',encoding='utf-8') as f2:
                            for line in lines:
                                if msg['Text'][7:] != line.strip():
                                    f2.write(line)
            if '清空words' in msg['Text']:
                with open('./words.txt','w',encoding='utf-8') as f:
                    f.flush()
            if '停止机器人' in msg['Text']:
                isrun='已停止！'
            if '启动机器人' in msg['Text']:
                isrun='运行中......'
            if '查看机器人' in msg['Text']:
                itchat.send(isrun, toUserName='filehelper')
            if 'robot' in msg['Text']:
                itchat.send(use_info, toUserName='filehelper')
```
- **定义监控撤回消息的函数（别人撤回的消息都会发到文件传输助手中）**

```python
@itchat.msg_register(NOTE,isFriendChat=True,isGroupChat=True)
def get_note(msg):
    if '撤回了一条消息' in msg['Text']:
        if '你撤回了一条消息' in msg['Text']:
            return None
        new_id=re.findall('<msgid>(\d+)</msgid>',str(msg))[0]
        public_time=time.strftime('%Y-%m-%d %H:%M:%S',time.gmtime(msg['CreateTime']+28800))
        data=msg_dict[new_id]
        nickname=re.findall("'UserName': '@[a-zA-Z0-9]+', 'NickName': '(.*?)', 'HeadImgUrl'",str(msg))[0]
        if len(nickname) > 100:
            friends_name=get_friendname()
            #群名这句我觉得还会有bug，由于测试的时候只在2个人的群中测试，多个的话可能会出bug，这个后面再说吧，反正我24小时挂着，有bug咱再说哈哈
            qun=re.findall("'NickName': '.*",re.findall("'UserName': '@[a-zA-Z0-9]+', 'NickName': '(.*)', 'HeadImgUrl'",str(msg))[0][-100:])[0].replace("'NickName': '",'')
            an=msg['ActualNickName']
            nickname='{0}在{1}群中'.format(an,qun)
            if msg['ActualUserName'] in friends_name.keys():
                if friends_name[msg['ActualUserName']]['remarkname'] != '':
                    nickname='{0}({1})在{2}群中'.format(an,friends_name[msg['ActualUserName']]['remarkname'],qun)
                else:
                    nickname='{0}({1})在{2}群中'.format(an,friends_name[msg['ActualUserName']]['nickname'],qun)
        if data['type'] == MAP:
            itchat.send('%s在%s撤回了一个位置，其位置：\n%s'%(nickname,public_time,data['data']),toUserName='filehelper')
        elif data['type'] == CARD:
            itchat.send('%s在%s撤回了一个名片，名片信息：\n%s，其头像：'%(nickname,public_time,data['data']),toUserName='filehelper')
            itchat.send('%s'%(data['head']),toUserName='filehelper')
        elif data['type'] == SHARING:
            itchat.send('%s在%s撤回了一个分享，其链接：\n%s'%(nickname,public_time,data['data']),toUserName='filehelper')
        elif data['type'] == TEXT:
            itchat.send('%s在%s撤回了一个信息，其内容：\n%s'%(nickname,public_time,data['data']),toUserName='filehelper')
        elif data['type'] in [PICTURE,RECORDING,ATTACHMENT,VIDEO]:
            itchat.send('%s在%s撤回了一张图片或者一个表情或者一段语音或者一个视频又或者一个文件，如下：'%(nickname,public_time),toUserName='filehelper')
            itchat.send('@%s@%s'%({'Picture': 'img', 'Video': 'vid'}.get(data['type'], 'fil'),data['file']),toUserName='filehelper')
```
- **到这里已经定义好了全部所需要的函数了，接下来就是文件的创建和表情包的收集，目前表情包是手动发表情让程序自动保存下来，其实可以定义一个添加表情的函数的，这个我后面会做出来，所以先这样吧，收集自定义表情包的函数如下：**

*ps：这个函数要另外单独运行（亲测商城里的表情包是保存不了的）*
```python
@itchat.msg_register(PICTURE) #只需要注册图片消息类型就可以了
def download_msg(msg):
    global q  #给文件传输助手发送你想要保存的表情包即可
    if msg['ToUserName'] == 'filehelper':
        msg['Text']('./表情包/' + str(q) + '.' + msg['FileName'].split('.')[-1])
        q+=1
q=1
itchat.auto_login(hotReload=True)
if not os.path.exists('./表情包'):
	os.makedirs('./表情包')
itchat.run()
```

### 三、思维导图（逻辑结构）
*ps：单看代码不过瘾的话看下面的图片吧，机器人处理消息的大致流程如下：*
![](https://img-blog.csdnimg.cn/20200103100316564.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM5NjI5MzIz,size_16,color_FFFFFF,t_70)
### 四、部署到云服务器
前面我有说过就是一定要`python3.7`版本的原因就在这里（也不是非要3.7版本，不过我敢肯定的是3.4版本是一定不行。）因为我本机上的就是3.7，可是服务器上的系统自带的是`python3.4`，然后如果你直接用3.4版本来运行，是可以运行的，只是返回来的`msg`是乱序的，每一次登录它都不一样，这样为什么不行呢，因为代码里面用了正则匹配，每次返回来的信息顺序都不一样的话，是没办法确定正则表达式的。这个坑坑了我一天好像，因为当时我就差这一步就完成了！想到会不会是`json`、`xml`版本的原因啊，然后这些又都是标准库，那么会不会是`python`版本原因造成的呢，于是乎，结果真的是这么回事！！说到云服务器，我之前的文章就有介绍过了，我用的是三丰云服务器，土豪请无视。

- 一样首先要在`linux`系统下先将自定义的表情包给上传了，这里推荐一个命令：`rz`   ，若还没有安装的，可以在终端运行如下代码，成功之后，进去想要上传的文件夹路径输入`rz`命令，会弹出选择文件的框，这时就可以把表情包全部上传了。

```bash
sudo apt-get install lrzsz
```

- 还有需要用到`screen`命令，这个是可以在`linux`系统上创建多个窗口，终端的很好用的命令，要用它来运行我们的微信机器人，这个应该可以在后面用在重登的情况上，因为各个窗口是互不影响的。

```bash
screen python3.7 robot.py
```

扫码登录成功之后，就可以开始属于你的个人微信机器人啦！！
ps：经过几天的验证，可以一直挂着，不会掉线，我怀疑那个心跳机制是假的？？这还需要经过时间的验证才能下定论。

### 五、运行展示
![](https://img-blog.csdnimg.cn/2019123016085447.gif)
*ps：由于gif有点难弄，就只先展示处理个人消息的吧，还有哪些处理群聊啊、给文件传输助手发指令啊，那些就不一一展示了，有兴趣的自己去实践就行了哈哈*
###### 参考链接
[https://www.php.cn/xiaochengxu-364486.html](https://www.php.cn/xiaochengxu-364486.html)
[https://itchat.readthedocs.io/zh/latest/](https://itchat.readthedocs.io/zh/latest/)
[https://blog.csdn.net/enweitech/article/details/79585043](https://blog.csdn.net/enweitech/article/details/79585043)

# 写在最后
没想到一开始从[itchat教程](https://itchat.readthedocs.io/zh/latest/tutorial/tutorial0/)的入门案例中，陷进去了，觉得挺好玩的，可是好景不长，就是他提供的那个图灵测试的key，竟然调用次数没有了，于是乎，便开始了属于自己的个人微信号自动回复机器人之旅，一开始也只是处理个人消息，后面就不断地加需求，才有了现在的`robot1.0`，其实我想实现的功能远不止这些：自动回复+监控撤回消息。其实我想到的还有：

 1. 自动回复中可以把语音识别给加进去，别人发语音也能跟他们交流；
 2. 不调用别人的接口，自己用`NLP`来实现一个自己的聊天机器人，让人工智障更加趋向人工智能；
 3. 退出重登，因为我看到有个参数是控制退出后会执行的函数，应该可以通过那个来实现，这个我就等看有没有心跳机制再搞吧；
 4. 实现建群拉人，找出那些删除了你的人；
 5. ==通过微信机器人控制电脑，通过手机发送指令就可以操作电脑，从而实现更多的功能；==
 6. 未完待续……
