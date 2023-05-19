### 由于微信视频号没有开放平台,无法通过正常的途径获取到个人微信视频号数据,所有通过爬取数据的方式

### 微信视频号后台管理地址

#### https://channels.weixin.qq.com/login.html


### 爬取流程
1.解析登陆二维码图片的真实地址 得到例如(https://channels.weixin.qq.com/mobile/confirm_login.html?token=AQAAAOagpYoBwf-LMqwVlg)

2. 通过F12,发现二维码真实地址的token参数是由 (https://channels.weixin.qq.com/cgi-bin/mmfinderassistant-bin/auth/auth_login_code?token=AQAAAO9KG4I-fsgWayCg7) 接口返回

3. 如何判断此二维码是否扫描状态,接口: https://channels.weixin.qq.com/cgi-bin/mmfinderassistant-bin/auth/auth_login_status?token=AQAAAP0W70L5h5ZcdGGnqw&timestamp=1684481485761&_log_finder_uin=&_log_finder_id=&scene=7&reqScene=7

4. 如何获取扫码用户的视频: https://channels.weixin.qq.com/cgi-bin/mmfinderassistant-bin/post/post_list (必须携带)Cookie，Cookie信息需要在
auth_login_code接口的响应头中获取