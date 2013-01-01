Backup your rails db with backup gem
============================================
Database backup is very vital once your application is in production. Nasty things can happen and it can turn into disaster if you loose your data.

Here i will walk you through taking db backup using ```backup``` gem and automating backup using ```whenever``` gem.

There are [handful](https://www.ruby-toolbox.com/categories/backups) of gems available to achieve this. But [backup](https://github.com/meskyanichi/backup) is most active of them and packed with features. To list out some them:

* Database Support [mysql, postgresql, mongoDB, redis, Riak]
* Storage Support [Amazon S3, Rackspace cloud files, Drop box and [lots more](https://github.com/meskyanichi/backup/wiki/Storages)]
* Compressors Support [Gzip, Bzip2, Pbzip2, Lzma]
* Notifications [Mail, Twitter, Campfire and what not. [Check all here](https://github.com/meskyanichi/backup/wiki/Notifiers])]
* Ruby support: 1.9.3, 1.9.2, 1.8.7

It has great [README](https://github.com/meskyanichi/backup#readme) and [wiki](https://github.com/meskyanichi/backup/wiki). I highly recommend to read them.

How to configure backup?
------------------------
Backup has handy generator to generate config file. Where you setup which database to take backup, where to take backup, where to notify when backup done, how to compress backup and many things.

So let's start. 

We'll be using following setup for our rails app:
* Database: Postgresql
* Backup storage: Amazon S3
* Notification: mail
* Compressor: gzip
* Encryptor: openssl

add backup gem to your Gemfile. 

```
gem backup
```
```
bundle install
```

Now run the generator provided by backup to generate config file.

```
$ cd /path/to/rails/app
```
```
$ bundle exec backup generate:model --trigger rails_db_backup --databases="postgresql" --storages="s3" --encryptors="openssl" --compressors="gzip" --notifiers="mail"
```

This will generate two config file at ```~/Backup/``` (the default location). One is a main configuration file where you can add common configuration for all your models. And another config is located at ```~/Backup/models/``` directory. In our case it will be ```~/Backup/models/rails_db_backup.rb```. Models are ```backup``` way of defining different backup configuration. So you can run them independently or in combination with each other. 

Let us see the model configuration file. Main configuration doesn't do much than loading all model configs by default.

File will look like this

```
Backup::Model.new(:rails_db_backup, 'Description for rails_db_backup') do

  split_into_chunks_of 250

  database PostgreSQL do |db|
    db.name               = "sweetly_us_development"
    db.username           = "kokanidivyesh"
    db.password           = ""
    db.host               = "localhost"
    # db.port               = 5432
    # db.socket             = "/tmp/pg.sock"
    # db.skip_tables        = ["skip", "these", "tables"]
    # db.only_tables        = ["only", "these" "tables"]
    # db.additional_options = ["-xc", "-E=utf8"]
    # Optional: Use to set the location of this utility
    #   if it cannot be found by name in your $PATH
    # db.pg_dump_utility = "/opt/local/bin/pg_dump"
  end

  #
  # checkout http://aws.amazon.com/s3 for more info
  #
  store_with S3 do |s3|
    s3.access_key_id     = "my_access_key_id"
    s3.secret_access_key = "my_secret_access_key"
    s3.region            = "us-east-1"
    s3.bucket            = "my-bucket" 
    s3.path              = "/backups"
    s3.keep              = 10
  end

  encrypt_with OpenSSL do |encryption|
    encryption.password      = "my_password"            # From String
    # encryption.password_file = "/path/to/password/file" # Or from File
    encryption.base64        = true
    encryption.salt          = true
  end

  compress_with Gzip

  notify_by Mail do |mail|
    mail.on_success           = true
    mail.on_warning           = true
    mail.on_failure           = true

    mail.from                 = "no-reply@sweetly.us"
    mail.to                   = "kunalchaudhari@email.com"
    mail.address              = "smtp.gmail.com"
    mail.port                 = 587
    mail.domain               = "sweetly.us"
    mail.user_name            = "no-reply@sweetly.us"
    mail.password             = "$weetly10"
    mail.authentication       = "plain"
    mail.enable_starttls_auto = true
  end

end
```

Let see one by one:

split_into_chunks_of line will tell to split the backup in chunk if size is greater than 250MB. This is helpful when you have large backup and you want to split it. For example Amazon allows max 5GB of data to be uploaded to S3.

It has handy methods for defining configuration for database, storage, notification and encryption. All are self-explanatory. Checkout above file where i have filled all relevant details.

By default, Backup will look for this file in ```~/Backup/config.rb```. If you want to place your configuration files in a different location, use the --config_file option: ```bundle exec backup perform --trigger my_backup --config_file '/path/to/config.rb'```

```bundle exec backup generate:model``` has various options you can specify to generate file at custom locations. ```backup help generate:model``` will give you all info.

How backup Works with above configuration?
------------------------------------------

* First, it will dump the PostgreSQL using pg_dump.
* Than PostgreSQL dump will be piped through the Gzip Compressor into rails_db_backup/databases/PostgreSQL/sweetly_us_development.sql.gz. 
* Finally, the rails_db_backup directory will be packaged into an uncompressed tar archive, which will be piped through the OpenSSL Encryptor to encrypt this final package into YYYY-MM-DD-hh-mm-ss.rails_db_backup.tar.enc. 
* This final encrypted archive will then be transferred to your Amazon S3 account. 
* If all goes well, and no exceptions are raised, you'll be notified via the mail notifier that the backup succeeded. 

If any warnings were issued or there was an exception raised during the backup process, then you'd receive an email in your inbox containing detailed exception information.

Running the Backup
------------------

So now lets push the button. Here is command to start/perform your backup.

```
cd path/to/your/rails_app
'''
'''
bundle exec backup perform -t rails_db_backup
```
