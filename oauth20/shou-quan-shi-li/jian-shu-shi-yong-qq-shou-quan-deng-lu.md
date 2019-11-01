# 简书使用QQ授权登录\(授权码模式\)

回顾一下前面提到的几个专有名词 , 这里面主要包含四个角色

1. Client - 需要授权的客户端 , 本文中就是【简书】
2. Resource Owner - 资源所有者 , 你可能会以为是 QQ , 但要想清楚 , QQ是属于个人的 , 所以在本文中资源所有者是指【QQ用户】
3. Authorization Server - 认证服务器 , 本文中特指【QQ互联平台】
4. Resource Server - 资源服务器 , 用来专门保存资源的服务器 , 接受通过访问令牌进行访问 , 这里特指【QQ用户信息中心】

#### 第一步 : 引导用户到认证服务器

打开目标网站简书 , 跳转到登录界面 , 要求用户登录 . 可是我们并未在简书注册帐号 , 所以就点击了QQ图标 , 准备使用QQ帐号进行集成登录 . 跳转到QQ登录界面后 , QQ要求用户授权 .

这一步中简书主要做了这样一件事就是引导用户到认证服务器 . 很显然【QQ互联平台】就是认证服务器 .

如何引导 ? 当然是页面跳转 .  
那认证服务器如何知道是简书过来的认证请求 ?  
当然是传参 . 那需要传递哪些参数呢 ? 回顾一下前面提到的相关参数 :

* response\_type：表示授权类型，必选项，此处的值固定为"code"
* client\_id：表示客户端的ID，必选项
* redirect\_uri：表示重定向URI，可选项
* scope：表示申请的权限范围，可选项
* state：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值。

```
https://graph.qq.com/oauth/show
?which=Login
&display=pc
&client_id=100410602
&redirect_uri=http%3A%2F%2Fwww.jianshu.com%2Fusers%2Fauth%2Fqq_connect%2Fcallback
&response_type=code
&state=c90638ba2b40716dbfc248f0109257bfb6cdb7da549b18bf
```

除了scope参数外，其他四个参数均有传参。此时你可能唯一对state参数比较迷惑，传递一个state参数，认证服务器会原封不动返回，那还干嘛要传递state参数呢 ?

这里可以理解为 , 简书用一个guid加长版字符串来唯一标识一个授权请求。这样才会正确获取授权服务器返回的授权码。

这里可能会让你产生疑惑，既然我知道了这些参数，我岂不是可以伪造简书认证请求，修改`redirect_uri`参数跳转到个人的网站，然后不就可以获取QQ授权？QQ互联平台在收到简书的授权请求时肯定会验证回调Url的 , 简书在QQ互联平台申请时肯定已经预留备案了要跳转返回的URL 。

#### 第二步 : 用户同意为第三方客户端授权

这一步，对于用户来说，只需要使用资源所有者（QQ）的用户名密码登录，并同意授权即可。点击授权并登录后，授权服务器首先会post一个请求回服务器进行用户认证，认证通过后授权服务器会生成一个授权码，然后服务器根据授权请求的`redirect_uri`进行跳转，并返回授权码`code`和授权请求中传递的`state`。

这里要注意的是：

**授权码有一个短暂的时效**

最终跳转回简书的Url为 :

```
http://www.jianshu.com/users/auth/qq_connect/callback
?code=093B9307E38DC5A2C3AD147B150F2AB3 
&state=bb38108d1aaf567c72da0f1167e87142d0e20cb2bb24ec5a
```

和之前的授权请求URL进行对比，可以发现`redirect_uri`、`state`完全一致。而`code=093B9307E38DC5A2C3AD147B150F2AB3`就是返回的授权码。

#### 第三步 : 使用授权码向认证服务器申请令牌

从这一步开始，对于用户来说是察觉不到的。简书后台默默的在做后续的工作。

简书拿到QQ互联平台返回的授权码后，需要根据授权码再次向认证服务器申请令牌（access token）。

到这里有必要再理清两个概念 :

* 授权码（Authorization Code）：相当于授权服务器口头告诉简书，用户同意授权使用他的QQ登录简书了。
* 令牌（Access Token）：相当于临时身份证。

那如何申请令牌呢 ? 简书需要后台发送一个get请求到认证服务器（QQ互联平台）, 要携带以下参数 :

* **grant\_type** - 表示授权类型，此处的值固定为"authorization\_code"，必选项；
* **client\_id** - 表示从QQ互联平台申请到的客户端ID，用来标志请求的来源，必选项；
* **client\_secret** - 这个是从QQ互联平台申请到的客户端认证密钥，机密信息十分重要，必选项；
* **redirect\_uri** - 成功申请到令牌后的回调地址；
* **code** - 上一步申请到的授权码。

根据以上信息我们可以模拟一个申请AccessToken的请求 :

```
https://graph.qq.com/oauth2.0/token
?client_id=100410602 
&client_secret=123456jianshu 
&redirect_uri=http://www.jianshu.com/users/auth/qq_connect/callback 
&grant_type=authorization_code 
&code=093B9307E38DC5A2C3AD147B150F2AB3
```

发送完该请求后，认证服务器验证通过后就会发放令牌，并跳转会简书，其中应该包含以下信息 :

* **access\_token** - 令牌
* **expires\_in** - access token的有效期，单位为秒。
* **refresh\_token** - 在授权自动续期步骤中，获取新的Access\_Token时需要提供的参数。

同样，我们可以模拟出一个返回的token :

```
http://www.jianshu.com/users/auth/qq_connect/callback
?access_token=548ADF2D5E1C5E88H4JH15FKUN51F 
&expires_in=36000
&refresh_token=53AD68JH834HHJF9J349FJADF3
```

这个时候简书还有一件事情要做，就是把用户token写到cookie里，进行用户登录状态的维持 . 例如 , 把用户token保存在名为`remember_user_token`的cookie里 .

#### 第四步 : 向资源服务器申请资源

有了token , 向资源服务器提供的资源接口发送一个get请求 , 资源服务器校验令牌无误 , 就会向简书返回资源\(QQ用户信息\) .

同样 , 来模拟一个使用token请求QQ用户基本信息资源的URL :

```
https://graph.qq.com/user/get_user_info
?client_id=100410602 
&qq=2098769873 
&access_token=548ADF2D5E1C5E88H4JH15FKUN51F
```

到这一步OAuth2.0的流程可以说是结束了 , 当然最后简书会存储**token、reresh\_token**和**用户数据这些重要的数据** .

#### 第五步 : 令牌延期（刷新）

第三步除了返回了token还有一个refresh\_token , 它是用来对令牌进行延期（刷新）的。为什么会有两种说法呢，因为可能认证服务器会重新生成一个令牌，也有可能对过期的令牌进行延期。

比如说，QQ互联平台为了安全性考虑，返回的`access_token`是有时间限制的，假如用户某天不想授权了呢，总不能给了个`access_token`你几年后还能用吧。我们上面模拟返回的令牌有效期为10小时。10小时后，用户打开浏览器逛简书，浏览器中用户的token对应的cookie已过期。简书发现浏览器没有携带token信息过来，就明白token失效了，需要重新向认证平台申请授权。如果让用户再点击QQ进行登录授权，这明显用户体验不好。这时`refresh_token`就派上了用场，可以直接跳过前面申请授权码的步骤，当发现token失效了，简书从浏览器携带的cookie中的sessionid找到存储在数据库中的`refresh_token`，然后再使用`refresh_token`进行token续期（刷新）。

那用refresh\_token进行token续期需要怎么做呢？同样 , 也需要向认证服务器发送一个get请求 , 参数有 :

* **grant\_type** - 表示授权类型，此处的值固定为"refresh\_token"，必选项
* **client\_id** - 表示从QQ互联平台申请到的客户端ID，用来标志请求的来源，必选项
* **client\_secret** - 这个是从QQ互联平台申请到的客户端认证密钥，机密信息十分重要，必选项
* **refresh\_token** - 即申请令牌返回的refresh\_token

我们可以再模拟一个令牌刷新的URL :

```
https://graph.qq.com/oauth2.0/token
?client_id=100410602 
&client_secret=123456jianshu 
&redirect_uri=http://www.jianshu.com/users/auth/qq_connect/callback 
&grant_type=refresh_token 
&refresh_token=53AD68JH834HHJF9J349FJADF3
```

这里返回的结果 , 和第四步中的结果一样 . 当然 , 这里的refresh\_token也不是无限延期的 , 也是有过期时间的 , 过期时间具体是由认证服务器决定的 . 一般来说`refresh_token`的过期时间要大于`access_token`的过期时间 ,

举个例子 :

假设简书从QQ互联平台默认获取到的`access_token`的有效期是1天，`refresh_token`的有效期为一周。用户今天使用QQ登录授权后，过了两天再去逛简书，简书发现token失效，立马用`refresh_token`刷新令牌，默默的完成了授权的延期。假如用户隔了两周再去逛简书，简书一核对，`access_token`、`refresh_token`全都失效，就只能乖乖引导用户到授权页面重新授权，也就是回到OAuth2.0的第一步。

