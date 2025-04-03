# Jupyterhub Installation Tutorial

## Section 1: Install and Configure WSL

**Open Powershell in administrator mode and run:**\
`wsl --install`

**Restart your machine**

**Open Powershell in user mode and run:**\
`wsl --set-default-version 2`

**Open Ubuntu from the start menu**

**Install Ubuntu**\
`wsl --install -d Ubuntu-24.04`\
*This command will take several minutes to complete*

**Launch Ubuntu from the start menu**

**Create a default Unix user account**

**Update and Upgrade**\
`sudo apt update && sudo apt upgrade`

## Section 2: Install and Configure Anaconda

**Install Anaconda**\
`wget https://repo.anaconda.com/archive/Anaconda3-2024.10-1-Linux-x86_64.sh`\
`bash ./Anaconda3-2024.10-1-Linux-x86_64.sh`\
`source ~/.bashrc`

**Create Conda Virtual Environment**\
`conda create -n <name of env> python=3.12`

**Activate Conda Virtual Environment**\
`conda activate <name of env>`

## Section 3: Install and Configure JupyterHub

**Install JupyterHub**\
`conda install jupyterhub`\
`conda install jupyterlab notebook`

**Install NodeJS and NPM**\
`sudo apt-get install nodejs npm`

**Install Configurable Http Proxy**\
`sudo npm install -g configurable-http-proxy`

**Install SudoSpawner via pip**\
`pip install sudospawner`

**Edit sudoers file**\
`sudo visudo`

Change this line:
```
secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

To this line:
```
secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/home/ai-class-test/anaconda3/envs/ai_test/bin"
```

Then add this block of text towards the bottom of the file:
```
Cmnd_Alias JUPYTER_CMD = /home/ai-class-test/anaconda3/envs/ai_test/bin/sudospawner
ALL ALL=(ALL:ALL) NOPASSWD:JUPYTER_CMD
```

**Generate and update JupyterHub config file**\
`jupyterhub --generate-config`

Add this block of text to your jupyterhub_config.py file:

```
c.Authenticator.allow_all = True

c.JupyterHub.spawner_class='sudospawner.SudoSpawner'

c.Spawner.default_url = '/lab'
```

## Section 4: Configure Connection to Microsoft SQL Server

**Create the bash scripting file**\
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

**Run this command**\
`sudo bash ./msodbc18.sh`

**Install tree**\
`sudo apt update && sudo apt upgrade`\
`sudo apt install tree`

`cd /opt`\
`ls`\
`tree`\
`cat microsoft/msodbcsql18/etc/odbcinst.ini`

`cd /etc`\
`ls -la | grep 'odbc`\
`sudo nano odbc.ini`

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

## Section 5: Test JupyterHub with a notebook

**Change to your home directory**\
`cd ~`

**Create a symbolic link**\
`sudo ln -s /home/ai-class-test/anaconda3/envs/ai_test/bin/jupyterhub /usr/local/bin/jupyterhub`

**Verify the link**\
`ls -l /usr/local/bin/jupyterhub`

**Start JupyterHub**\
`sudo jupyterhub`

**Login with your UNIX credentials**

**Choose Notebook -> Python 3 (ipykernel)**

**Rename the notebook**\
`conda install pyodbc pandas numpy`

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

**Run the Jupyter notebook**
