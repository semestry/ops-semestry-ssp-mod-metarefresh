Automated Metadata Management
=============================

<!-- 
	This file is written in Markdown syntax. 
	For more information about how to use the Markdown syntax, read here:
	http://daringfireball.net/projects/markdown/syntax
-->


<!-- {{TOC}} -->

Introduction
------------

If you want to connect an Identity Provider, or a Service Provider to a **federation**, you need to setup metadata for the entries that you trust. In many federations, in particular federations based upon the Shibboleth software, it is normal to setup automated distribution of metadata using the SAML 2.0 Metadata XML Format.

Some central administration or authority, provides a URL with a SAML 2.0 document including metadata for all entities in the federation.

The present document explains how to setup automated downloading and parsing of a metadata document on a specific URL.



Preparations
------------

You need to enable the following modules:

 1. [cron](./cron:cron)
 2. metarefresh

The cron module allows you to do tasks regularly, by setting up a cron job that calls a hook in SimpleSAMLphp.

The metarefresh module will download and parse the metadata document and store it in metadata files cached locally.

First, you will need to copy the `config-templates` files of the two modules above into the global `config/` directory.

	[root@simplesamlphp] cd /var/simplesamlphp
	[root@simplesamlphp simplesamlphp] touch modules/cron/enable
	[root@simplesamlphp simplesamlphp] cp modules/cron/config-templates/*.php config/
	[root@simplesamlphp simplesamlphp] touch modules/metarefresh/enable
	[root@simplesamlphp simplesamlphp] cp modules/metarefresh/config-templates/*.php config/



Testing it manually
-------------------

It is often useful to verify that the metadata sources we want to use can be parsed and verified by metarefresh, before actually
configuring it. We can do so in the command line, by invoking metarefresh with the URL of the metadata set we want to check. For
instance, if we want to configure the metadata of the SWITCH AAI Test Federation:

	cd modules/metarefresh/bin
	./metarefresh.php -s http://metadata.aai.switch.ch/metadata.aaitest.xml

The `-s` option sends the output to the console (for testing purposes). If the output makes sense, continue. If you get a lot of error messages, try to read them and fix the problems that might be causing them. If you are having problems and you can't figure out the cause, you can always send an e-mail to the SimpleSAMLphp mailing list and ask for advice.



Configuring the metarefresh module
----------------------------------


Now we are going to proceed to configure the metarefresh module. First, edit the appropriate configuration file:


	[root@simplesamlphp simplesamlphp]# vi config/config-metarefresh.php

Here's an example of a possible configuration for both the Kalmar Federation and UK Access Management Federation:

	$config = [
		'sets' => [
			'kalmar' => [
				'cron'		=> ['hourly'],
				'sources'	=> [
					[
						'src' => 'https://kalmar.feide.no/simplesaml/module.php/aggregator/?id=kalmarcentral&mimetype=text/plain&exclude=norway',
						'certificates' => [
							'current.crt',
							'rollover.crt',
						],
						'template' => [
							'tags'	=> ['kalmar'],
							'authproc' => [
								51 => ['class' => 'core:AttributeMap', 'oid2name'],
							],
						],
					],
				],
				'expireAfter' 		=> 60*60*24*4, // Maximum 4 days cache time.
				'outputDir' 	=> 'metadata/metarefresh-kalmar/',
				'outputFormat' => 'flatfile',
			],
			'uk' => [
				'cron'		=> ['hourly'],
				'sources'	=> [
					[
						'src' => 'http://metadata.ukfederation.org.uk/ukfederation-metadata.xml',
					],
				],
				'expireAfter' 		=> 60*60*24*4, // Maximum 4 days cache time.
				'outputDir' 	=> 'metadata/metarefresh-ukaccess/',
				'outputFormat' => 'serialize',
			],
		]
	];


The configuration consists of one or more metadata sets. Each metadata set has its own configuration, representing a metadata set of sources.
Some federations will provide you with detailed instructions on how to configure metarefresh to fetch their metadata automatically, like,
for instance, [the InCommon federation in the US](https://spaces.internet2.edu/x/eYHFAg). Whenever a federation provides you with specific
instructions to configure metarefresh, be sure to use them from the authoritative source.

The metarefresh module supports the following configuration options:

`cron`
:   Which cron tags will refresh this metadata set.

`sources`
:   An array of metadata sources that will be included in this
    metadata set. The contents of this option will be described later in more detail.

`expireAfter`
:   The maximum number of seconds a metadata entry will be valid.

`outputDir`
:   The directory where the generated metadata will be stored. The path
    is relative to the SimpleSAMLphp base directory.

`outputFormat`
:   The format of the generated metadata files. This must match the
    metadata source added in `config.php`.

`types`
:	The sets of entities to load. An array containing strings identifying the different types of entities that will be
	loaded. Valid types are:

	* saml20-idp-remote
	* saml20-sp-remote
	* attributeauthority-remote

	All entity types will be loaded by default.

Each metadata source has the following options:

`src`
:   The source URL where the metadata will be fetched from.

`certificates`
:   An array of certificate files, the filename is relative to the `cert/`-directory,
    that will be used to verify the signature of the metadata. The public key will
    be extracted from the certificate and everything else will be ignored. So it is
    possible to use a self signed certificate that has expired. Add more than one
    certificate to be able to handle key rollover.

`template`
:   This is an array which will be combined with the metadata fetched to
    generate the final metadata array.

`types`
:	Same as the option with the same name at the metadata set level. This option has precedence when both are specified,
	allowing a more fine grained configuration for every metadata source.

`whitelist`
:   This is an array that allows for selectively refreshing a list of identity providers. Only data from identity
    providers that exist in the whitelist will be processed and written to disk. This is especially useful for hosting
    environments that have strict limits memory and maximum execution time.

`blacklist`
:   This is an array that allows for selectively skipping a list of identity providers.  Only data from identity
    providers that do not appear in the blacklist are processed and written to disk.


After you have configured the metadata sources, you need to give the
web-server write access to the output directories. Following the previous example:

	chown www-data /var/simplesamlphp/metadata/metarefresh-kalmar/
	chown www-data /var/simplesamlphp/metadata/metarefresh-ukaccess/

Now you can configure SimpleSAMLphp to use the metadata fetched by metarefresh. Edit the main
config.php file, and modify the `metadata.sources` directive accordingly: 

	'metadata.sources' => [
		['type' => 'flatfile'],
		['type' => 'flatfile', 'directory' => 'metadata/metarefresh-kalmar'],
		['type' => 'serialize', 'directory' => 'metadata/metarefresh-ukaccess'],
	],

Remember that the `type` parameter here must match the `outputFormat` in the configuration of the module.



Configuring the cron module
---------------------------

See the [cron module documentation](./cron:cron) to configure `cron`

Once you have invoked cron, and if this operation seems to run fine, navigate to the **SimpleSAMLphp Front page** › **Federation**. Here you will see a list of all the Identity Providers trusted. They will be listed with information about the maximum duration of their cached version, such as *(expires in 96.0 hours)*.

You *may* need to adjust the below php.ini setings if the metadata files you consume are quite large.

* `memory_limit`
* `max_execution_time`

Metadata duration
-----------------

SAML metadata may supply a `cacheDuration` attribute which indicates the maximum time to keep metadata cached. Because this module is run from cron, it cannot decide how often it is run and enforce this duration on its own. Make sure to run metarefresh from cron at least as often as the shortest `cacheDuration` in your metadata sources.
