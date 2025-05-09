# Jupyterhub Installation Tutorial

## Section 1: Install and Configure WSL

1. Open Powershell in administrator mode and run:\
`wsl --install`

2. Restart your machine

3. Open Powershell in user mode and run:\
`wsl --set-default-version 2`

4. Launch Ubuntu from the start menu

5. Create a default Unix user account

6. Update and Upgrade\
`sudo apt update && sudo apt upgrade`

<br>

## Section 2: Install and Configure Anaconda

1. Install Anaconda\
`wget https://repo.anaconda.com/archive/Anaconda3-2024.10-1-Linux-x86_64.sh`\
`bash ./Anaconda3-2024.10-1-Linux-x86_64.sh`\
`source ~/.bashrc`

	OPTIONAL: Install Miniconda
	```
	wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
	bash ./Miniconda3-latest-Linux-x86_64.sh
	source ~/miniconda3/bin/activate
	conda init
	conda config --remove channels defaults
	conda config --add channels conda-forge
	conda config --set channel_priority strict
	```

3. Create a Conda Virtual Environment\
`conda create -n <name of env> python=3.12`

4. Activate the Conda Virtual Environment\
`conda activate <name of env>`

<br>

## Section 3: Install and Configure JupyterHub

1. Install JupyterHub\
`conda install jupyterhub`\
`conda install jupyterlab notebook`

2. Install NodeJS and NPM\
`sudo apt-get install nodejs npm`

3. Install Configurable Http Proxy\
`sudo npm install -g configurable-http-proxy`

4. Install SudoSpawner via pip\
`pip install sudospawner`

5. Edit the sudoers file\
`sudo visudo`

	Change this line:
	```
	secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
	```

	To this line:
	```
	secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/home/<username>/anaconda3/envs/<name of env>/bin"
	```

	Then add this block of text towards the bottom of the file:
	```
	Cmnd_Alias JUPYTER_CMD = /home/<username>/anaconda3/envs/<name of env>/bin/sudospawner
	ALL ALL=(ALL:ALL) NOPASSWD:JUPYTER_CMD
	```

6. Generate and update a JupyterHub config file\
`jupyterhub --generate-config`

	Add this block of text to your jupyterhub_config.py file:
	```
	c.Authenticator.allow_all = True
	
	c.JupyterHub.spawner_class='sudospawner.SudoSpawner'
	
	c.Spawner.default_url = '/lab'
	```

<br>

## Section 4: Configure Connection to Microsoft SQL Server

1. Create a bash scripting file named msodbc18.sh in the user's home directory\
`touch msodbc18.sh`

	Add this block of text to the newly created file:
	```
	if ! [[ "20.04 22.04 24.04 24.10" == *"$(grep VERSION_ID /etc/os-release | cut -d '"' -f 2)"* ]]; then
		echo "Ubuntu $(grep VERSION_ID /etc/os-release | cut -d '"' -f 2) is not currently supported.";
		exit;
	fi
	
	# Download the package to configure the Microsoft repo
	curl -sSL -O https://packages.microsoft.com/config/ubuntu/$(grep VERSION_ID /etc/os-release | cut -d '"' -f 2)/packages-microsoft-prod.deb
	# Install the package
	sudo dpkg -i packages-microsoft-prod.deb
	# Delete the file
	rm packages-microsoft-prod.deb
	
	# Install the driver
	sudo apt-get update
	sudo ACCEPT_EULA=Y apt-get install -y msodbcsql18
	# optional: for bcp and sqlcmd
	sudo ACCEPT_EULA=Y apt-get install -y mssql-tools18
	echo 'export PATH="$PATH:/opt/mssql-tools18/bin"' >> ~/.bashrc
	source ~/.bashrc
	# optional: for unixODBC development headers
	sudo apt-get install -y unixodbc-dev
	```

2. Run the bash script\
`sudo bash ./msodbc18.sh`

	OPTIONAL: Install tree
	```
	sudo apt update && sudo apt upgrade
	sudo apt install tree
	tree /opt
	cat microsoft/msodbcsql18/etc/odbcinst.ini
	```

3. Edit the odbc.ini file\
`ls -la /etc | grep 'odbc'`\
`sudo nano /etc/odbc.ini`

	Add this block of text:
	```
	[Downpower_POC]
	Driver=ODBC Driver 18 for SQL Server
	Server=GAXGPSQ123DV
	Database=Downpower_POC
	UID=downpower_db
	PWD=<redacted> (replace with your db password)
	Port=1433
	Encrypt=yes
	TrustServerCertificate=yes
	
	[DownPowerPOC_DW]
	Driver=ODBC Driver 18 for SQL Server
	Server=GAXGPSQ123DV
	Database=DownPowerPOC_DW
	UID=downpower_db
	PWD=<redacted> (replace with your db password)
	Port=1433
	Encrypt=yes
	TrustServerCertificate=yes
	```

<br>

## Section 5: Test JupyterHub with a notebook

1. Install a few python data science packages via Anaconda
`conda install pyodbc pandas numpy`

2. Create a symbolic link to shorten the 'sudo jupyterhub' command\
`sudo ln -s /home/<username>/anaconda3/envs/<name of env>/bin/jupyterhub /usr/local/bin/jupyterhub`\
`ls -l /usr/local/bin/jupyterhub`

4. Start JupyterHub\
`sudo jupyterhub`

5. Login with your UNIX credentials

6. Choose Notebook -> Python 3 (ipykernel)

7. Rename the notebook

8. Add these blocks of code to individual cells in the new notebook

	```
	import pyodbc
	import pandas as pd
	import numpy as np
	from IPython.display import display  # For Jupyter Notebook display
	```

	```
	# Databricks settings
	
	# Ensure all columns are displayed
	pd.set_option('display.max_columns', None)  # Show all columns
	#pd.set_option('display.width', 500)  # Prevent line wrapping
	#pd.set_option('display.max_colwidth', None)  # Show full content in each cell
	pd.set_option('display.max_colwidth', 50)
	#pd.set_option('display.max_rows', 100) #Show All Rows Without Limit
	```

	```
	%%time
	
	server = 'gaxgpsq123dv'
	database = 'DownPowerPOC_DW'
	username = 'downpower_db'
	password = '<redacted>'
	connection_string = f'DRIVER={{ODBC Driver 18 for SQL Server}};TRUSTSERVERCERTIFICATE=Yes;DSN=Downpower_POC;SERVER={server};DATABASE={database};UID={username};PWD={password}'
	
	# Establish connection
	try:
	    conn = pyodbc.connect(connection_string)
	    print("Connected to SQL Server successfully!")
	except Exception as e:
	    print("Error connecting to SQL Server:", e)
	
	non_weather_source_data_query1 = """ 
	SELECT *
	FROM [Downpower_POC].[dbo].[PI_Data_Prior_2024]
	WHERE (CONVERT(DATE, Timestamp) >= '2021-01-01' AND CONVERT(DATE, Timestamp) <= '2022-06-30')
	AND Tag in (SELECT distinct Source_Tag
	FROM [Downpower_POC].[dbo].[Tag_Mapping]
	WHERE Name NOT LIKE ('%weather%'));
	"""
	non_weather_source_data_query2 = """ 
	SELECT *
	FROM [Downpower_POC].[dbo].[PI_Data_Prior_2024]
	WHERE (CONVERT(DATE, Timestamp) >= '2022-07-01' AND CONVERT(DATE, Timestamp) <= '2023-12-31')
	AND Tag in (SELECT distinct Source_Tag
	FROM [Downpower_POC].[dbo].[Tag_Mapping]
	WHERE Name NOT LIKE ('%weather%'));
	"""
	
	#non_weather_src_df1 = pd.read_sql(non_weather_source_data_query1, conn)
	#non_weather_src_df2 = pd.read_sql(non_weather_source_data_query2, conn)
	
	non_weather_fact_data_query = """ 
	SELECT distinct Unit, System, PITagName, EquipmentId, PIMeanValue, PITimeStamp
	FROM [DownPowerPOC_DW].[dbo].[FactPIData]
	ORDER BY PITimeStamp;
	"""
	
	non_weather_fact_df = pd.read_sql(non_weather_fact_data_query, conn)
	
	tag_mapping_query = """SELECT distinct *
	  FROM [Downpower_POC].[dbo].[Tag_Mapping]
	  WHERE Name NOT LIKE ('%weather%')"""
	
	tag_mapping_df = pd.read_sql(tag_mapping_query, conn)
	
	fact_downpower_query = """
	SELECT [DownPowerEventType],[CauseCode],[Unit],[IsForced],[DownPowerEventStartTime],[DownPowerEventEndTime],CONVERT(DATE, DownPowerEventStartTime) as DownPowerEventStartDate, 
	CONVERT(DATE, DownPowerEventEndTime) as DownPowerEventEndDate,[DownPowerEventDurationHours],[DownPowerEventCapacityReduction],[DownPowerEventGenerationLoss],[DownPowerEventDescription]
	FROM [DownPowerPOC_DW].[dbo].[FactDownPower]
	WHERE IsForced = 'Y';
	"""
	
	fact_downpower_df = pd.read_sql(fact_downpower_query, conn)
	
	#Close connection
	conn.close()
	
	print("Command completed successfully")
	```

	```
	# Display results
	print("Number of rows and columns:", non_weather_fact_df.shape)
	non_weather_fact_df.head()
	```

9. Run the Jupyter notebook

<br>
