---
layout: post
title: PHP的Realpath Cache
city: 南京
tags: [tech]
---

###前言

PHP的缓存有很多种，包括输出缓冲([ob系列函数][1])，opcode缓存([APC][2]，[eAccelerator][3]，[XCache][4]等扩展实现)，这些大家已经很熟悉了，接下来介绍一下一个不太被人注意的PHP缓存机制：Realpath Cache。

###介绍

**require**，**require\_once**，**include**，**include\_once**这四个语句(并非函数)大家经常会用到，如果用这类语句去包含文件(相对路径)的话，那么PHP会去[include_path][5]所指定的路径中去查找相关文件。一个应用中会存在大量的require\_once语句调用，如果每次调用都去include\_path中查找相应的文件，势必会对应用的性能产生负面影响。为了避免这种负面效应产生的影响，PHPER们会使用文件的绝对路径来包含所需的文件，这样就减少了查询include\_path的次数。 其实，PHP自5.1.0起，就引入了[RealpathCache][6]。RealpathCache可以把PHP所用到文件的[realpath][7]进行缓存，以便PHP再使用这些文件的时候不需要再去include_path中查找，加快PHP的执行速度。

###配置

realpath cache的[配置项][8]有两个，分别为**realpath_cache_size**和**realpath_cache_ttl**，可以在php.ini中进行修改：

	; Determines the size of the realpath cache to be used by PHP. This value should
	; be increased on systems where PHP opens many files to reflect the quantity of
	; the file operations performed.
	; http://php.net/realpath-cache-size
	;realpath_cache_size = 16k

	; Duration of time, in seconds for which to cache realpath information for a given
	; file or directory. For systems with rarely changing files, consider increasing this
	; value.
	; http://php.net/realpath-cache-ttl
	;realpath_cache_ttl = 120

其中realpath\_cache\_size指定了realpath cache的大小，默认为16k，如果你觉得这个容量太小，可以适当增加；realpath\_cache\_ttl指定了缓存的过期时间，默认为120秒，对于不经常修改的生产环境来说，这个数字可以调整的更大些。

###问题

由于realpath会展开symlink(即软连接)，所以如果你使用修改symlink目标这种方式发布应用的新版本的话，realpath cache会导致一些问题的出现：当你修改symlink使其指向一个新的release目录时候，由于realpath cache所缓存内容还没有过期，于是就会出现应用使用的还是旧的release，直到realpath cache所缓存内容过期失效为止(默认120秒)，或者重启php-fpm。 

看个例子： 
基础环境：nginx + fastcgi + php-fpm    
应用环境：/var/www/app是一个symlink，并做为document_root，在/var/www下存在version0.1，version0.2两个版本的release。     
初始情况下/var/www/app指向version0.1  

	lrwxr-xr-x    1 weizhifeng  staff    10 10 22 16:41 app -> version0.1
	drwxr-xr-x    3 weizhifeng  staff   102 10 22 16:43 version0.1
	drwxr-xr-x    3 weizhifeng  staff   102 10 22 16:43 version0.2

version0.1，version0.2内部各有一个hello.php

	$ cat version0.1/hello.php 
	<?php 
	echo 'in version0.1';
	?>

	$ cat version0.2/hello.php 
	<?php 
	echo 'in version0.2';
	?>

nginx配置文件片段：

	location / {
	    root /var/www/app;   #app为symlink
	    index  index.php index.html index.htm;
	}

	location ~ .php$ {
	    root /var/www/app; #app为symlink
	    fastcgi_pass   127.0.0.1:9000;
	    fastcgi_index  index.php;
	    include        fastcgi_params;
	}

此时通过HTTP访问hello.php，得到的内容是「in version0.1」；修改/var/www/app，使其指向version0.2

	$ rm -f app && ln -s version0.2 app

修改完成之后通过HTTP访问hello.php，得到的内容仍旧是「in version0.1」，可见是realpath cache在作祟了，此时你可以重启php-fpm或者等待120秒钟让realpath cache失效。 你可以使用[clearstatcache][9]来清除realpath cache，但是这个只对当前调用clearstatcache函数的PHP进程有效，而其他的PHP进程还是无效，由于PHP进程池（php-fpm生成，或者Apache在prefork模式下产生的N个httpd子进程）的存在，这个方法不是很适用。

参考：

[http://php.net/manual/en/ini.core.php#ini.sect.performance](http://php.net/manual/en/ini.core.php#ini.sect.performance)
[http://sixohthree.com/1517/php-and-the-realpath-cache](http://sixohthree.com/1517/php-and-the-realpath-cache)

[1]: http://cn.php.net/manual/en/ref.outcontrol.php "outcontrol"
[2]: http://php.net/manual/en/book.apc.php "apc"
[3]: http://sourceforge.net/projects/eaccelerator/ "eaccelerator"
[4]: http://xcache.lighttpd.net/ "xcache"
[5]: http://cn2.php.net/manual/zh/ini.core.php#ini.include-path "include path"
[6]: http://php.net/manual/en/ini.core.php#ini.sect.performance "Real path Cache"
[7]: http://cn2.php.net/manual/en/function.realpath.php "realpath"
[8]: http://us2.php.net/manual/en/ini.core.php#ini.realpath-cache-size "realpath cache 配置"
[9]: http://us3.php.net/clearstatcache "clearstatcache"