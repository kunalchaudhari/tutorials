Backup your rails app db with backup gem
========================================
Database backup is very vital once your application is in production. Nasty things can happen and it can turn into disaster if you loose your data.

Here i will walk you through taking db backup using ```backup``` gem.

There are [handful](https://www.ruby-toolbox.com/categories/backups) of gems available to achieve this. But [backup](https://github.com/meskyanichi/backup) is most active of them and packed with features. To list out some them:

* Database Support [mysql, postgresql, mongoDB, redis, Riak]
* Storage Support [Amazon S3, Rackspace cloud files, Drop box and [lots more](https://github.com/meskyanichi/backup/wiki/Storages)]
* Compressors Support [Gzip, Bzip2, Pbzip2, Lzma]
* Notifications [Mail, Twitter, Campfire and what not. [Check all here](https://github.com/meskyanichi/backup/wiki/Notifiers)]
* Ruby support: 1.9.3, 1.9.2, 1.8.7

It has great [README](https://github.com/meskyanichi/backup#readme) and [wiki](https://github.com/meskyanichi/backup/wiki). I highly recommend to read them.


So let's start. Our requirement are as follow:

* We will be taking backup for our Postgresql db
* Compress db backup with gzip
* Encrypt db backup with openssl
* Store that backup on Amazon S3
* Notified via email when backup done

First of all add backup gem to your Gemfile.

```
gem backup
```
```
$ bundle install
```

Now run the generator provided by backup to generate config file.

```
$ cd /path/to/rails/app
```
```
$ bundle exec backup generate:model --trigger rails_db_backup \
              --databases="postgresql" --storages="s3" \
              --encryptors="openssl" --compressors="gzip" --notifiers="mail"
```

This will generate two config file at ```~/Backup/``` (the default location.). One is a main configuration file which has common configuration for all your models. This file will load other configurations from ```~/Backup/models/```. In our case it will be ```~/Backup/models/rails_db_backup.rb```. Main configuration doesn't do much than loading all model configs by default.

Note: Models are ```backup``` way of defining different backup configuration. So you can run them independently or in combination with each other. 

Let's have a look at ```~/Backup/models/rails_db_backup.rb``` configuration file. 

```
Backup::Model.new(:rails_db_backup, 'Description for rails_db_backup') do

  split_into_chunks_of 4000

  database PostgreSQL do |db|
    db.name               = "sweetly_us_production"
    db.username           = "app"
    db.password           = "secret"
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
    s3.access_key_id     = "SKIAKL97GE7ZI3ZCXM2A"
    s3.secret_access_key = "DSJUAW9n8rQNIRw8EgXlQai6qZfcr6rYnipGCjFq"
    s3.region            = "us-east-1"
    s3.bucket            = "sweetlyus.dbbackups" 
    s3.path              = "/backups"
    s3.keep              = 10
  end

  encrypt_with OpenSSL do |encryption|
    encryption.password      = "secret"            # From String
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
    mail.password             = "secret"
    mail.authentication       = "plain"
    mail.enable_starttls_auto = true
  end

end
```

It has handy methods for defining configuration for database, storage, notification and encryption. All are self-explanatory. Checkout above file where i have filled all relevant details.

```split_into_chunks_of``` will tell to split the backup in chunk if size is greater than 4GB. This is helpful when you have large backup and you want to split it. For example, Amazon allows max 5GB chunk to be uploaded to S3.

By default, Backup will look for this file in ```~/Backup/config.rb```. You can place your configuration files in a different location. If you place file at different location use the --config_file option: ```bundle exec backup perform -t my_backup --config_file '/path/to/config.rb'```

```bundle exec backup generate:model``` has various options you can specify to generate file at custom locations. ```backup help generate:model``` will give you all info.

How backup Works with above configuration?
------------------------------------------

* First, it will dump the PostgreSQL using pg_dump.
* Than PostgreSQL dump will be piped through the Gzip Compressor into ~/Backup/.tmp/rails_db_backup/databases/PostgreSQL/sweetly_us_development.sql.gz. 
* Finally, the rails_db_backup directory will be packaged into tar archive, which will be piped through the OpenSSL Encryptor to encrypt this final package into YYYY-MM-DD-hh-mm-ss.rails_db_backup.tar.enc. 
* This final encrypted archive will then be transferred to your Amazon S3 account. 
* If all goes well, and no exceptions are raised, you'll be notified via the mail notifier that the backup succeeded. 

If any warnings were issued or there was an exception raised during the backup process, then you'd receive an email in your inbox containing detailed exception information.

Running the Backup
------------------

So now lets push the button. Here is command to start/perform your backup.

```
cd path/to/your/rails_app
```
```
bundle exec backup perform -t rails_db_backup
```

That's it.

Logs
----
Backup keeps a log with information about all the backups it performs. The log contains various informational messages, any warnings that may be issued and any errors which occur during the process. By default, the log file is stored in the ```~/Backup/log``` directory. If you would like to store the log for your backup in another location, then specify an alternate ````--log-path``` on the command line:

For all other options you can pass to ```backup perform``` can be found here: [https://github.com/meskyanichi/backup/wiki/Performing-Backups](https://github.com/meskyanichi/backup/wiki/Performing-Backups).

I have touched only little about backup here. Checkout [https://github.com/meskyanichi/backup](https://github.com/meskyanichi/backup) for all available features.

Automatic backups with whenever gem
-----------------------------------
Since Backup is an easy-to-use command line utility, you should write a crontask to invoke it periodically. There is a nice gem available to achieve. You can checkout here: [https://github.com/javan/whenever](https://github.com/javan/whenever)

Conclusion
----------
Backup is gem that enables you to easily perform backup operations. It has built-in support for various databases, storage protocols/services, syncers, compressors, encryptors and notifiers which you can mix and match. For example, With the [Syncers](https://github.com/meskyanichi/backup/wiki/Syncers) you can transfer directories of data from the production server to the backup server. This is extremely useful to take backup of your images, music, videos or other heavy file formats.
