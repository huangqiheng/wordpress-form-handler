wordpress-form-handler
======================

监视nginx产生的log文件，解释出所有wordpress的form请求并记录和分析
同时将最新提交的内容，同步到其他业务系统中

一、安装必须的软件包
-----------------------------

	apt-get install ruby ruby-dev
	apt-get install build-essential
	apt-get install mysql-server mysql-client
	apt-get install libmysqlclient-dev
	apt-get install gearman


二、安装ruby gems
----------------------------

	在采集端：
	gem install gearman-ruby

	在处理端：
	gem install gearman-ruby
	gem install wp_rpc
	gem install mysql2
	gem install activerecord

三、配置nginx采集日志
----------------------------

1、在nginx.conf的http中填入：

        log_format form_tracking '{'
                '"time_local" : "$time_local", '
                '"server_name" : "$server_name", '
                '"request_uri" : "$request_uri", '
                '"http_referer" : "$http_referer", '
                '"status" : $status, '
                '"request_length" : $request_length, '
                '"http_cookie" : "$http_cookie", '
                '"request_body" : "$request_body"'
                '}';

2、在nginx.conf所在目录中，新创建以下内容的文件：

	#创建这个文件
	vim form_log_patch

	#在这个文件中填入如下内容
	set $form_flag '';
	if  ($request_method = POST) {
		set $form_flag 1$form_flag;
	}

	if ($content_type = 'application/x-www-form-urlencoded') {
		set $form_flag 2$form_flag;
	}

	if ($http_referer) {
		set $form_flag 3$form_flag;
	}

	if  ($form_flag = '321') {
		access_log "/var/log/nginx/wordpress.form.log" form_tracking;
		access_log on;
	}


3、然后在各server中，改写php的location中新添一行：

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass unix:/var/run/php5-fpm-appgame.sock;
                fastcgi_index index.php;
                include fastcgi_params;
		#这是新添的一行
                include form_log_patch;
        }

