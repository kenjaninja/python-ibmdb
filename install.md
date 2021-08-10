# Installing Python ibm_db on z/OS

### Prerequisites:
	
	zODBC(64 bit) installed with z/OS 2.3
	IBM Python 3.8.3 64 bit
	Install the below PTFs (if not installed):
		UI72588 (v11)
		UI72589 (v12)

## Configuring the environment
1. Configure an appropriate _Db2 ODBC initialization file_ that can be read at application time. For compatibility with ibm_db, the following properties must be set:

    In COMMON section:

    ```ini
    MULTICONTEXT=2
    CURRENTAPPENSCH=ASCII
    FLOAT=IEEE
    ```

    In SUBSYSTEM section:

    ```ini
    MVSATTACHTYPE=RRSAF
    ```

    Here is a sample of a complete initialization file:

    ```ini
    ; This is a comment line...
    ; Example COMMON stanza
    [COMMON]
    MVSDEFAULTSSID=VC1A
    CONNECTTYPE=1
    MULTICONTEXT=2
    CURRENTAPPENSCH=ASCII
    FLOAT=IEEE
    ; Example SUBSYSTEM stanza for VC1A subsystem
    [VC1A]
    MVSATTACHTYPE=RRSAF
    PLANNAME=DSNACLI
    ; Example DATA SOURCE stanza for STLEC1 data source
    [STLEC1]
    AUTOCOMMIT=1
    CURSORHOLD=1
    ```

    Reference [Db2 ODBC initialization file](https://www.ibm.com/docs/en/db2-for-zos/12?topic=applications-db2-odbc-initialization-file) in IBM Docs for more details.

1. Configure environment variables for install/compilation or create a shell profile (i.e. `.profile` file in your home environment) which includes environment variables needed for Python and DB2 for z/OS ODBC (make sure the paths are changed based on your system and DB setting and the variables are configured).

	```sh
	# Code page autoconversion i.e. USS will automatically convert between ASCII and EBCDIC where needed.
	export _BPXK_AUTOCVT='ON'
	export _CEE_RUNOPTS='FILETAG(AUTOCVT,AUTOTAG) POSIX(ON) XPLINK(ON)'
	
	# Python
	export PYTHON_HOME=/path/to/python/install
	export PATH=$PYTHON_HOME/bin:$PATH
	export LIBPATH=$PYTHON_HOME/lib:$PATH
	
	export DSNAOINI=/path/to/odbc_XXXX.ini	# Db2 ODBC initialization file
	
	# Set IBM_DB_HOME to the High Level Qualifier (HLQ) of your Db2 datasets.
	# For example, if your Db2 datasets are located as XXXX.DSN.VC10.SDSNC.H and XXXX.DSN.VC10.SDSNMACS, 
	# you need to set the IBM_DB_HOME variable to XXXX.DSN.VC10
	export IBM_DB_HOME=XXXX.DSN.VC10
	export STEPLIB=$STEPLIB:$IBM_DB_HOME.SDSNEXIT:$IBM_DB_HOME.SDSNLOAD:$IBM_DB_HOME.SDSNLOD2
	
	# In case you have the SDSNC.H data set named anything other than SDSNC.H, i.e. non default behaviour, 
	# configure following variable. IGNORE OTHERWISE.
	# export DB2_INC=$IBM_DB_HOME.XXXX.H
	
	# In case you have the SDSNMACS data set named anything other than SDSNMACS, i.e. non default behaviour, 
	# configure following variable. IGNORE OTHERWISE.
	# export DB2_MACS=$IBM_DB_HOME.XXXX
	```
1. To validate your python install, run `python3 -V`. Should return `Python 3.8.3` or greater.
1. Unless you are a sysprog, you will likely not have authority to install packages globally, so consider creating a python virtual environment:
	```sh
	python3 -m venv $HOME/ibm_python_venv
	source $HOME/ibm_python_venv/bin/activate
	```
4. Make sure `pip3` is installed and working as part of the Python installation.


## Installing Python ibm_db and running a validation Program

Now that the Python and ODBC is ready, we need `ibm_db` to connect to DB2.

1. Run `pip3 install ibm_db` to install ibm_db: 
1. ODBC connects and works with the DB2 for z/OS on the same subsytem or Sysplex using details configured in the Db2 ODBC `.ini` file . No additional settings or credentials are needed during connection creation in the python script, as shown in the example below:
	```python
	import ibm_db
	conn = ibm_db.connect('', '', '')
	```
1. Run the test program below to validate setup: `python3 test.py`
	
	**test.py**

	```python
	from __future__ import print_function
	import sys
	import ibm_db

	print('ODBC test start')
	conn = ibm_db.connect('', '', '')
	if conn:
		stmt = ibm_db.exec_immediate(conn, "SELECT CURRENT DATE FROM SYSIBM.SYSDUMMY1")
		if stmt:
			while (ibm_db.fetch_row(stmt)):
				date = ibm_db.result(stmt, 0)
				print("Current date is:", date)
		else:
			print(ibm_db.stmt_errormsg())
		ibm_db.close(conn)
	else:
		print("No connection:", ibm_db.conn_errormsg())
	print('ODBC test end')
	```	

# Common issues
## Missing header files
```
Using legacy 'setup.py install' for ibm-db, since package 'wheel' is not installed.
Installing collected packages: ibm-db
    Running setup.py install for ibm-db ... error
...
    WARNING CCN3296 ./ibm_db.c:27    #include file <Python.h> not found.
    WARNING CCN3296 ./ibm_db.c:28    #include file <datetime.h> not found.
    WARNING CCN3296 ./ibm_db.h:16    #include file <Python.h> not found.
    WARNING CCN3296 ./ibm_db.h:17    #include file <structmember.h> not found.
...
    ERROR CCN3309 ./ibm_db.h:229   The offsetof macro cannot be used with an incomplete struct or union.
    ERROR CCN3277 ./ibm_db.h:229   Syntax error: possible missing ';' or ','?
    CCN0793(I) Compilation failed for file ./ibm_db.c.  Object file not created.
    error: command '/bin/xlc' failed with exit code 12
```
Your python is not a `python-dev` installation, which means it does not include necessary C header files used to install `ibm_db`. For Python 3.8, these header files usually reside in the `include/python3.8` of a `python-dev` installation directory.

### Solution
On z/OS, your python directory may have a `python-dev` installation along with your regular python installation. You can search for a relevant header file to find this directory.
```sh
ls /usr/lpp/python38/pyz/include/		# No header files in include directory
# 

find /usr/lpp/python38/ -name "Python.h"	# Find Python header file
# /usr/lpp/python38/usr/lpp/IBM/cyp/v3r8/pyz/include/python3.8/Python.h

# Use newly found python-dev install and retry install
export PYTHON_HOME=/usr/lpp/python38/usr/lpp/IBM/cyp/v3r8/pyz
export PATH=$PYTHON_HOME/bin:$PATH
export LIBPATH=$PYTHON_HOME/lib:$PATH
```

## ERROR: No .egg-info directory found in /tmp/...
`IBM_DB_HOME` environment variable is not set
### Solution
Set `IBM_DB_HOME` to the High Level Qualifier (HLQ) of your Db2 datasets (refer to environment variable configuration step in **Configuring the environment** section)
