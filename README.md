On the beanstalk
======================

### Step 1: Set up a new project

Make a new project folder and initialize the git repository:

````
mkdir myapp && cd myapp
git init
````


Add a baseline `.gitignore` file:

````
#in .gitignore...
.DS_Store
.idea
````

Add `index.php`. We'll keep it simple for now...

````
<?php
//in index.php...
echo 'hello world';
````


Initialize Elastic Beanstalk with `eb`. Instructions for installing `eb` [can be found here](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/command-reference-get-started.html). In addition to `eb`, you'll need to have an AWS account and create an AWS access key and secret before you begin.

````
eb init
````

Once you have entered your credentials, `eb` will ask you a series of questions. The answers important to this tutorial are below:

1. Environment Tier: select `WebServer::Standard::1.0`


2. Solution Stack: since WordPress is a PHP application, select one of the PHP stacks, like `64bit Amazon Linux 2013.09 running PHP 5.5`.

3. Environment Type: for the purposes of this tutorial, select `SingleInstance`. 

4. Create an RDS DB Instance? Type `y` for yes. This creates a database server that is separate from from your web server. Not only does this allow you to push code updates to your web server without wiping out your database, it also means that you can eventually have several web server instances accessing the single database server. 

5. Attach an instance profile: `Create a default instance profile`

At this point you'll notice that `eb` has added an `.elasticbeanstalk` folder to your project root and added it to `.gitignore`.

````
- myapp
  - .elaticbeanstalk
    - config
  - .gitignore
  - index.php

````

````
#in .gitignore...
.DS_Store
.idea
.elasticbeanstalk/
````

Commit your changes...

````
git add -A
git commit -m 'initial commit'
````

Start the app:

```
eb start
```

AWS takes about 5 minutes to create the resources for the first time. You can check the status of your deployment an the AWS control panel, or use this time to set up deployment keys for any private repos that you will be using.

### Step 2: Set up deployment keys

Elastic Beanstalk runs composer on each web server instance whenever you push a new version of your code, updating the dependencies you've defined in `composer.json`. For public, open source code, this is no problem. For private code repositories &mdash; i.e. your code that you do not want to make public &mdash; the web servers will need to authenticate with BitBucket or GitHub in order to access the repos.

You'll only need one key pair for each git host (BitBucket or GitHub.)  That is you may want to use several private repositories on BitBucket: each of those repositories should share the same key.  

For this example, we'll set up a deployment key for BitBucket.

Create a key pair on the commandline, naming it something descriptive like `bitbucket_deployment_key`, and using no passphrase:

```
ssh-keygen
````

Copy the public key to your clipboard:

```
pbcopy < bitbucket_deployment_key.pub
```

Add this public key to **each** of the private repositories you'll be using. On BitBucket, the page to do this is found at 

```
bitbucket.org/YOUR_ORG/YOUR_PRIVATE_REPO/admin/deploy-keys
````

Create a folder in your project root called `.ebextensions`. Create a file within that directory called `deploy.config`.  This is where we'll tell Beanstalk to add the private key. Use this template:

```
files:
  "/root/.ssh/bitbucket_deployment_key":
     mode: "000600"
     owner: root
     group: root
     content: |
       -----BEGIN RSA PRIVATE KEY-----
       PUT YOUR PRIVATE KEY HERE
       -----END RSA PRIVATE KEY-----
  "/root/.ssh/config":
       mode: "000600"
       owner: root
       group: root
       content: |
         Host bitbucket.org
            StrictHostKeyChecking no
            IdentityFile /root/.ssh/bitbucket_deployment_key
            UserKnownHostsFile /dev/null
```

...replacing...

```
       -----BEGIN RSA PRIVATE KEY-----
       PUT YOUR PRIVATE KEY HERE
       -----END RSA PRIVATE KEY-----
````

...with the contents of your private key.  You can copy the private key to your clipboard by doing this:

```
pbcopy < bitbucket_deployment_key
```

Note that `deploy.config` is YAML, so whitespaces matter. What we're going here is telling Elastic Beanstalk to add the `bitbucket_deployment_key` private key to `/root/.ssh/` and adding a `config` file to the same directory.  The `config` file tells `ssh` to use the key to authenticate with BitBucket. 

Somewhat importantly (and not very well-documented by AWS,) the commands run by Beanstalk servers when you deploy new code are run as `root`, not the ec2 user.

Now set up your composer dependencies in a `composer.json` file at the root of your project:

```
{
	"name" : "your-org/myapp",
	"description": "A WordPress app",
	"license" : "proprietary",
	"repositories": [
		{
			"type": "vcs",
			"url": "git@bitbucket.org:your-org/your-private-repo.git"
		}
	],
	"require": {
		"your-org/your-private-repo.git": "dev-master"
	}
}
```

Don't forget to replace `your-org/your-private-repo.git` with one of your own private repos.

Run composer:


```
composer update
```

This will add a `vendor` directory to your project root. You want to gitignore this, since Elastic Beanstalk will take care of installing your dependendencies.  Add `vendor` to `.gitignore`:


````
#in .gitignore...
.DS_Store
.idea
.elasticbeanstalk/
vendor/
````

Commit the changes, making sure to add `composer.json`, `composer.lock`, and the contents of `.ebextensions`.

```
git add -A
git commit -m 'added dependencies'

```

**Important:** By committing `.ebextensions` and it's contents, you're committing the contents of a private key. This is (at this point) a necessary evil, since Elastic Beanstalk uses `git` to push changes. But it does mean (1) that you shouldn't make the project a public repository, and (2) that your Bitbucket and GitHub deployment keys should **only be used for the purpose of deploying code to your servers.** 


###Step 3: Push your dependencies

**Recommended:** Check that your original app works first. Go to the [Elastic Beanstalk Console](https://console.aws.amazon.com/elasticbeanstalk) and check the status of your app. Click on the app's URL. Assuming you haven't pushed any new code since doing `eb start`, you should have a website that says "hello world".

Once you know that the baseline Hello World app is running, modify `index.php`:

```
<?php
ini_set('display_errors', true);
/**
 * use the Composer autoloader...
 */
require_once __DIR__ . '/vendor/autoload.php';

/**
 * Test whether our private code library has been installed...
 */

new YourOrg\YourLib\YourClass;


echo 'hello world. it worked.';
```

Now commit your changes and push them to Beanstalk with `git`.  This is as simple as:

```
git commit -a -m 'test whether our private library is installed'
git aws.push
```

Updating code with `aws.push` this way takes a lot less time than `eb start`. You can watch the progress of the update on the AWS console. When the update finishes, go to the events tab, and make sure there are no errors. If all is well &mdash; in this case, if composer was able to access your private code repo &mdash; you won't see any errors, and your app will work.

**Troubleshooting**

If there are errors, you can do a couple of things:

1. Check the logs. Go to the Logs tab and click Snapshot.  This is a quick way to `tail` the server logs without `ssh`ing into the server instance.

2. `ssh` into the server.  This requires a several steps. First, create a key pair in EC2 -> Key Pairs. From your app's Elastic Beanstalk console, edit Configuration -> Instances. Select the EC2 Key Pair you just created.  Saving this configuration will require a restart of the server.  Go to Services -> EC2 _. Instances and find your instance.  Find out what security group it belongs to, then make sure that the security group has port 22 open. Third, go back to EC2->Instances, select the instance, and click "Connect".  This will give you instructions for `ssh`ing into the server using the key pair. Once you're in:

```
sudo su
ls /root/.ssh
# make sure your private key and config files exist, etc
```

Once you're sure everything is hunky-dory, proceed to the next step.

### Step 4: Set up a local development environment

Make a local database:

    mysql -u root -p
    mysql> CREATE DATABASE `myapp.loc`  CHARACTER SET utf8 COLLATE utf8_general_ci;
    mysql> GRANT ALL PRIVILEGES ON `myapp.loc`.* TO 'webapp'@'localhost';


Make a virtual host. 

    sudo nano /etc/hosts

Add an entry for myapp.loc:

    sudo nano /etc/hosts
    
	##
	# Host Database
	#
	# localhost is used to configure the loopback interface
	# when the system is booting.  Do not change this entry.
	##
	127.0.0.1       localhost
	255.255.255.255 broadcasthost
	::1             localhost 
	fe80::1%lo0     localhost
	
	127.0.0.1       myapp.loc


Edit your apache virtual hosts. On my machine this is located at `/usr/local/etc/apache2/extra/httpd-vhosts.conf`.

    sudo nano /usr/local/etc/apache2/extra/httpd-vhosts.conf 

Add an entry for myapp.loc. It should look something like this:

````
<VirtualHost *:80>
    ServerAdmin you@example.com
    DocumentRoot "/Users/you/dev/myapp"
    ServerName myapp.loc
    DirectoryIndex index.php
    <Directory "/Users/you/dev/myapp">
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        Allow from all
    </Directory>
 </VirtualHost>
 ````
 
 Restart apache...
 
     sudo apachectl restart

...and make sure the app works at [myapp.loc](http://myapp.loc).

### Step 5: Add WordPress

In your project root:

Get the latest, unzip it, and move it to the root...

    curl http://wordpress.org/latest.tar.gz -o latest.tar.gz
    tar -zxvf latest.tar.gz
    mv wordpress/* ./
     

Delete the `latest.tar.gz` archive and `wordpress` directory...

     rm latest.tar.gz 
     rmdir wordpress/

We'll need `mu-plugins` later...

     mkdir wp-content/mu-plugins

`chown` wp-content locally and make it writable by the group, which should include the local apache user
     
     sudo chown -R chris:webdev wp-content/
     sudo chmod -R g+rwx wp-content/
     
Create a `wp-config.php` and get some unique salts...

    cp wp-config-sample.php wp-config.php  
    curl https://api.wordpress.org/secret-key/1.1/salt/


Edit `wp-config.php`. 

 - Replace the salt `define`s with the salt code you just generated.
 - Replace the database `define`s with a conditional statement:
 
````
if (isset($_SERVER['IS_DEV_LOCAL']) && 1 == $_SERVER['IS_DEV_LOCAL']){
	//local...
	define('DB_NAME', 'myapp.loc');
	define('DB_HOST', $_SERVER['DEV_DB_HOST']);
	define('DB_USER', $_SERVER['DEV_DB_USER']);
	define('DB_PASSWORD', $_SERVER['DEV_DB_PASS']);


} elseif (isset($_SERVER['RDS_DB_NAME'])){
	//AWS Environment
	define('DB_NAME', $_SERVER['RDS_DB_NAME']);
	define('DB_USER', $_SERVER['RDS_USERNAME']);
	define('DB_PASSWORD', $_SERVER['RDS_PASSWORD']);
	define('DB_HOST', $_SERVER['RDS_HOSTNAME'] . ':' . $_SERVER['RDS_PORT']);
		
}
````

Note that for this to work locally you need to add `IS_DEV_LOCAL`, `DEV_DB_HOST`, `DEV_DB_USER` and `DEV_DB_PASS` to your local environment variables. You can do this with the `SetEnv` apache directive at the bottom of your `httpd.conf` file.

	SetEnv IS_DEV_LOCAL 1
	SetEnv DEV_DB_HOST localhost
	SetEnv DEV_DB_USER webapp
	SetEnv DEV_DB_PASS SuperSecret123
	
On Elastic Beanstalk the `$_SERVER['RDS_DB_*']` are automatically set for you.

You can use this conditional statement in other useful ways, such as setting `WP_DEBUG` to be true on your local machine but false on the Beanstalk servers.

Finally, add one more line at the top of `wp-config.php`:

	require_once __DIR__ . '/vendor/autoload.php'
	
Your `wp-config.php` file should look something like this:

	<?php
	/**
	 * The base configurations of the WordPress.
	 *
	 * This file has the following configurations: MySQL settings, Table Prefix,
	 * Secret Keys, WordPress Language, and ABSPATH. You can find more information
	 * by visiting {@link http://codex.wordpress.org/Editing_wp-config.php Editing
	 * wp-config.php} Codex page. You can get the MySQL settings from your web host.
	 *
	 * This file is used by the wp-config.php creation script during the
	 * installation. You don't have to use the web site, you can just copy this file
	 * to "wp-config.php" and fill in the values.
	 *
	 * @package WordPress
	 */
	
	require_once __DIR__ . '/vendor/autoload.php';
	
	if (isset($_SERVER['IS_DEV_LOCAL']) && 1 == $_SERVER['IS_DEV_LOCAL']){
		//local...
		define('DB_NAME', 'myapp.loc');
		define('DB_HOST', $_SERVER['DEV_DB_HOST']);
		define('DB_USER', $_SERVER['DEV_DB_USER']);
		define('DB_PASSWORD', $_SERVER['DEV_DB_PASS']);
	
		define('WP_DEBUG', true);
		
	
	} elseif (isset($_SERVER['RDS_DB_NAME'])){
		//AWS Environment
		define('DB_NAME', $_SERVER['RDS_DB_NAME']);
		define('DB_USER', $_SERVER['RDS_USERNAME']);
		define('DB_PASSWORD', $_SERVER['RDS_PASSWORD']);
		define('DB_HOST', $_SERVER['RDS_HOSTNAME'] . ':' . $_SERVER['RDS_PORT']);
	
		define('WP_DEBUG', false);
	
	}
	
	define('AUTH_KEY',         '8Z|pa|z4+<Yid7M6}zYXWM^?-Z>M:QaFD>.W3j_-JHg244)z|yWXZ1CN+ (FAy^K');
	define('SECURE_AUTH_KEY',  '+wO<?[{]Pv1}0FP,%Z?XGRW`9WE4|*Lo$&2np{-.cB&0GF/5Isj_|sRn_]lM=Wg/');
	define('LOGGED_IN_KEY',    'L@SL%A#(JAoT|6#i8pEY!FCV%%|~t#GXxk,, >.gPmqIzF7VY][rrCj!B|+q+(6c');
	define('NONCE_KEY',        'F]T5]LJppsW3xl|NS! trXf-b#ny;n4wa^Ra}k|7g_D3UTq_<noZ@mz%)3P- 5V;');
	define('AUTH_SALT',        '+ft+=xh6/FiJS490+l<czw%D<,-zqeziQ)0AF7>CRNWFdJr -!IXPppIx oO,v0&');
	define('SECURE_AUTH_SALT', 'SzdgSC1P;8vtuXS<2^,d-mMxwz}5@Sm|-.8!-}/|U|q+v)gG8fDTXZIYWei//V];');
	define('LOGGED_IN_SALT',   'ien@<`4N)2yR*[1_dO8b|d+BrLx6^XPYE>Ws#8$]P]IOJOxBAu<.X?~vRqWMQ#]1');
	define('NONCE_SALT',       'X_=S^3+9OX}(451Pa9IDlT%p:c}>o9Jx.;;+~f%D]w|Z+s.2oiZouI2 =&lTFjnj');
	
	/**
	 * WordPress Database Table prefix.
	 *
	 * You can have multiple installations in one database if you give each a unique
	 * prefix. Only numbers, letters, and underscores please!
	 */
	$table_prefix  = 'wp_';
	
	/**
	 * WordPress Localized Language, defaults to English.
	 *
	 * Change this to localize WordPress. A corresponding MO file for the chosen
	 * language must be installed to wp-content/languages. For example, install
	 * de_DE.mo to wp-content/languages and set WPLANG to 'de_DE' to enable German
	 * language support.
	 */
	define('WPLANG', '');
	
	
	
	/* That's all, stop editing! Happy blogging. */
	
	/** Absolute path to the WordPress directory. */
	if ( !defined('ABSPATH') )
		define('ABSPATH', dirname(__FILE__) . '/');
	
	/** Sets up WordPress vars and included files. */
	require_once(ABSPATH . 'wp-settings.php');
















