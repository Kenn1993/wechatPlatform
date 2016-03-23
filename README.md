# 微信平台开发者模式：自定义菜单+自定义回复
[![Support](https://img.shields.io/badge/support-PHP-blue.svg?style=flat)](http://www.php.net/)
[![Support](https://img.shields.io/badge/support-ThinkPHP-red.svg?style=flat)](http://www.thinkphp.cn/)

## 原理介绍
微信公众平台后台能让用户自己设置自动回复和自定义菜单等功能，但微信自带的功能是有部分限制的。而启动开发者模式后就无限制了，但原有的自动回复和自定义菜单的设置就会失效，需要调用微信提供的一系列接口进行设置，这里将介绍如何设置自动回复和自定义菜单。微信接口文档:<a href="http://mp.weixin.qq.com/wiki/home/index.html">http://mp.weixin.qq.com/wiki/home/index.html</a>


## 实现流程
### 1、下载官方php事例文件   <a href="http://mp.weixin.qq.com/mpres/htmledition/res/wx_sample.20140819.zip">点击下载</a>
部分代码如下(我进行了一些修改使得更适合TP框架和业务逻辑):定义token的位置设置自己定义的token，下一步骤会使用到,上传该文件到服务器存放控制器的文件夹。
	
	/**
	 * 微信公众平台类
	 */
	class WechatPlatformAction extends CommonAction{
	    public function index(){
	        //定义 token
	        define("TOKEN", "weixin");
	        if(isset($_GET["echostr"])){
	            //调用验证方法
	            $this->valid();
	        }else{
	            //调用自动回复方法
	            $this->responseMsg();
	        }
	    }
	    /*
	     * 验证方法（第一次开启开发者模式时调用）
	     */
	    public function valid(){
	        $echoStr = $_GET["echostr"];
	        //valid signature , option
	        if($this->checkSignature()){
	            echo $echoStr;
	            exit;
	        }
	    }
		
	    private function checkSignature(){
	        // you must define TOKEN by yourself
	        if (!defined("TOKEN")) {
	            throw new Exception('TOKEN is not defined!');
	        }
	
	        $signature = $_GET["signature"];
	        $timestamp = $_GET["timestamp"];
	        $nonce = $_GET["nonce"];
	
	        $token = TOKEN;
	        $tmpArr = array($token, $timestamp, $nonce);
	        // use SORT_STRING rule
	        sort($tmpArr, SORT_STRING);
	        $tmpStr = implode( $tmpArr );
	        $tmpStr = sha1( $tmpStr );
	
	        if( $tmpStr == $signature ){
	            return true;
	        }else{
	            return false;
	        }
	    }
### 2、进入<a href="https://mp.weixin.qq.com/">微信公众平台后台</a>，点击左栏下方的“基本配置”。
   1. 进入该页面后，会看到AppID和AppSecret，这是调用大部分接口要用到的验证信息。
   2. 而下方则是“服务器配置”，点击修改配置，进入配置页。
   3. 这里会看到几个需要配置的地方，第一个是URL:这是步骤1事例文件的位置，我之所以修改成"控制器/方法"的形式，就是为了更好的调用，例如:kennblog.caterest.com/wechatPlatform/index。
   4. 第二个是Token:这个就是步骤1中，你已经定义好的TOKEN。
   5. 第三个是EncodingAESKey，这个直接点随机生成就好。
   6. 第四个是加解密模式，根据需要选择，开发时选兼容模式。
   7. 点击提交，如果成功证明该微信平台已经连接到你的服务器，回到上一个页面启用即可；如果失败，有可能是文件里的TOKEN和后台设置的Token不一致，或者URL不能正常访问。
### 3、设置自动回复
启用了开发者模式后，微信公众平台用户所发的信息都会以POST的形式直接发送到设置好的URL，即php事例文件，而php事例文件自动调用responseMsg方法，对发送类型进行判断并回复内容，我这里只展示了event（关注事件）和 text（发送文字）两种情况，不同类型有不同的xml输出格式，请根据需要<a href="http://mp.weixin.qq.com/wiki/1/6239b44c206cab9145b1d52c67e6c551.html">查询文档</a>。

<font color="red">注意：事例代码是没有执行responseMsg这个方法的，要自己添加上，另外它是用$GLOBALS["HTTP_RAW_POST_DATA"]来获取POST数据的，有些服务器会设置了不能用$GLOBALS进行获取，因此可以修改成file_get_contents("php://input")进行获取POST数据</font>
     
	
	/*
     * 自动回复
     */
    public function responseMsg(){
        $postStr = file_get_contents("php://input");
        if (!empty($postStr)){
            libxml_disable_entity_loader(true);
            $postObj = simplexml_load_string($postStr, 'SimpleXMLElement', LIBXML_NOCDATA);
            $fromUsername = $postObj->FromUserName;
            $toUsername = $postObj->ToUserName;
            $keyword = trim($postObj->Content);
            $msgType = $postObj->MsgType;
            $time = time();
            $textTpl = "<xml>
							<ToUserName><![CDATA[%s]]></ToUserName>
							<FromUserName><![CDATA[%s]]></FromUserName>
							<CreateTime>%s</CreateTime>
							<MsgType><![CDATA[%s]]></MsgType>
							<Content><![CDATA[%s]]></Content>
							<FuncFlag>0</FuncFlag>
							</xml>";
            if($msgType=='event'){//关注时自动回复
                $msgType = "text";
                $contentStr = "亲，感谢您关注kennblog";
                $resultStr = sprintf($textTpl, $fromUsername, $toUsername, $time, $msgType, $contentStr);
                echo $resultStr;
            }elseif($msgType=='text' && $keyword){
                $msgType = "text";
                $contentStr = "技术交流请到kennblog.caterest.com留言";
                $resultStr = sprintf($textTpl, $fromUsername, $toUsername, $time, $msgType, $contentStr);
                echo $resultStr;
            }else{
                echo "";
            }
        }else {
            echo "";
            exit;
        }
    }

当然，真实的业务逻辑并没有那么简单，正常逻辑下开发者需要根据msgType和keyword进行数据库查询，得出结果并进行输出。

### 4、设置自定义菜单
自定义菜单需要获取access_token，才能进行调用自定义菜单的一系列接口。

首先我们调用获取access_token的接口，并封装成一个方法，这里就需要步骤1中的AppID和AppSecret:

	/*
     * 获取access_token
     */
    private  function getAccessToken(){
        $AppID = 'xxxxxxxxxxx';
        $AppSecret ='xxxxxxxxxxxxxxx';
        $url = 'https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid='.$AppID.'&secret='.$AppSecret;
        //curl
        $curl = curl_init();
        curl_setopt($curl, CURLOPT_URL,$url);
        curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);
        curl_setopt($curl, CURLOPT_GET, 1);
        $result = curl_exec($curl);
        curl_close($curl);
        $result = json_decode($result,true);
        return $result['access_token'];
    }

封装后，由于原来的公众平台非开发者模式下已经设置了自定义菜单，我这里先调用了获取自定义菜单接口，再调用生成自定义菜单接口，这样就可以重现原有设置好的菜单了。<a href="http://mp.weixin.qq.com/wiki/10/0234e39a2025342c17a7d23595c6b40a.html">具体文档</a>

要注意的是这2个接口的数据格式都是json，与自定义回复的xml格式不同，而获取回来的数组格式与要发送的数组格式也是不同的，要进行处理才能正常生成自定义菜单。

	/*
     * 生成自定义菜单(获取非开发者模式下的菜单再生成一次)
     */
    public function createMenu(){
        //获取access_token
        $access_token = $this->getAccessToken();
        //获取原有菜单设置
        $url = 'https://api.weixin.qq.com/cgi-bin/get_current_selfmenu_info?access_token='.$access_token;
        //curl
        $curl = curl_init();
        curl_setopt($curl, CURLOPT_URL,$url);
        curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);
        curl_setopt($curl, CURLOPT_GET,true);
        $result = curl_exec($curl);
        curl_close($curl);
        $result = json_decode($result,true);
        //生成自定义菜单
        $postData=array();
        $postData['button'] = $result['selfmenu_info']['button'];
        //处理数据
        foreach($postData['button'] as $k=>$v){
            foreach($v['sub_button']['list'] as $k2=>$v2){
                if($v2['type']=='news'){
                    $v['sub_button']['list'][$k2]['type'] = 'view_limited';
                    $v['sub_button']['list'][$k2]['media_id'] = $v2['value'];
                    if($v2['news_info'] && $v2['value']){
                        unset($v['sub_button']['list'][$k2]['value']);
                        unset($v['sub_button']['list'][$k2]['news_info']);
                    }
                }
            }
            if($v['sub_button']){
                $postData['button'][$k]['sub_button'] = $v['sub_button']['list'];
            }
        }
        $json = json_encode($postData,JSON_UNESCAPED_UNICODE);
        $url = 'https://api.weixin.qq.com/cgi-bin/menu/create?access_token='.$access_token;
        //curl
        $curl = curl_init();
        curl_setopt($curl, CURLOPT_URL,$url);
        curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);
        curl_setopt($curl, CURLOPT_POST,1);
        curl_setopt($curl, CURLOPT_POSTFIELDS, $json);
        $result = curl_exec($curl);
        curl_close($curl);
        $result = json_decode($result,true);
        dump($result);
    }#   w e c h a t P l a t f o r m  
 