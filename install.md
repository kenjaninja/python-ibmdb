# Installing Python ibm_db on z/OS

### Prerequisites:
	
	zODBC(64 bit) installed with z/OS 2.3
	IBM Python 3.8.3 64 bit
	Install the below PTFs (if not installed):
		UI72588 (v11)
		UI72589 (v12)

## Configuring the environment
1. Configure environment variables for install/compilation or create a shell profile (i.e. `.profile` file in your home environment) which includes environment variables needed for Python and DB2 for z/OS ODBC (make sure the paths are changed based on your system and DB setting and the variables are configured).

	```
	# Code page autoconversion i.e. USS will automatically convert between ASCII and EBCDIC where needed.
	export _BPXK_AUTOCVT='ON'
	export _CEE_RUNOPTS='FILETAG(AUTOCVT,AUTOTAG) POSIX(ON) XPLINK(ON)
	export PATH=$HOME/bin:/user/python_install/bin:$PATH
	export LIBPATH=$HOME/lib:/user/python_install/lib:$PATH
	
	export DSNAOINI=$HOME/odbc_XXXX.ini	# Db2 ODBC initialization file
	
	# Set IBM_DB_HOME to the High Level Qualifier (HLQ) of your Db2 datasets.
	# For example, if your Db2 datasets are located as XXXX.DSN.VC10.SDSNC.H and XXXX.DSN.VC10.SDSNMACS, 
	# you need to set IBM_DB_HOME variable to XXXX.DSN.VC10
	export IBM_DB_HOME=XXXX.DSN.VC10
	export STEPLIB=$STEPLIB:$IBM_DB_HOME.SDSNEXIT:$IBM_DB_HOME.SDSNLOAD:$IBM_DB_HOME.SDSNLOD2
	
	# In case you have the SDSNC.H data set named anything other than SDSNC.H, i.e. non default behaviour, 
	# configure following variable. IGNORE OTHERWISE.
	# export DB2_INC=$IBM_DB_HOME.XXXX.H
	
	# In case you have the SDSNMACS data set named anything other than SDSNMACS, i.e. non default behaviour, 
	# configure following variable. IGNORE OTHERWISE.
	# export DB2_MACS=$IBM_DB_HOME.XXXX
	```
1. To validate python install, run `python3 -V`. Should return `Python 3.8.3` or greater.
1. Unless you are a sysprog, you will likely not have authority to install packages globally, so consider creating a python virtual environment:
	```
	python3 -m venv $HOME/ibm_python_venv
	source $HOME/ibm_python_venv/bin/activate
	```
4. Make sure `pip3` is installed and working as part of the Python installation.


## Installing Python ibm_db and running a validation Program

Now that the Python and ODBC is ready, we need `ibm_db` to connect to DB2.

1. Run `pip3 install ibm_db` to install ibm_db: 
1. ODBC connects and works with the DB2 for z/OS on the same subsytem or Sysplex using details configured in the Db2 ODBC `.ini` file . No additional settings or credentials are needed during connection creation in the python script, as shown in the example below:
	```
	import ibm_db
	conn = ibm_db.connect('', '', '')
	```
1. Run the test program below to validate setup: `python3 test.py`
	
	**test.py**

	```
	from __future__ import print_function
	import sys
	import ibm_db

	print('ODBC Test start')
	conn = ibm_db.connect('', '', '')
	if conn:
		stmt = ibm_db.exec_immediate(conn, "SELECT CURRENT DATE FROM SYSIBM.SYSDUMMY1")
		if stmt:
			while (ibm_db.fetch_row(stmt)):
				date = ibm_db.result(stmt, 0)
				print("Current date is :", date)
		else:
			print(ibm_db.stmt_errormsg())
		ibm_db.close(conn)
	else:
		print("No connection:", ibm_db.conn_errormsg())
	print('ODBC Test end')
	```	
