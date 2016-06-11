# Command Line Semantics and Use Cases

Use case: Keeping in progress backups locally for reverting

Scenario: While on the road, you routinely work on a set of files, some of them very large. The baseline copies of these files are backed up at home, but sometimes in the field you’ll make changes and want to be able to go back and look at previous versions. These files are together 1 GB, so your backup directory will start at 1 GB.

First, you’ll want to create a local baseline

	gfs vol create ~/backups/ vol:local-backups:
	# Sets up ~/backups as a place to hold local backups

Now, tell GFS you want to use local-backups as the default location for backups

	gfs backup default vol:local-backups

Now on a periodic basis, you’ll want to run an incremental backup. When you run this next command, anything in ~/work-dir that is not present in the local-backup store will be saved away. 

	gfs backup run ~/work-dir

How do you get back your files? The simplest way is just to pen up ~/backups. Inside, you’ll find directories like this:

