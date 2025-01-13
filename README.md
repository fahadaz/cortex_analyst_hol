# Cortex Analyst HOL
Cortex Analyst HOL with Notebook, Snowpark Pandas and Streamlit in Snowflake app

Please run follwing script to setup for HOL

```
-------SETUP SCRIPT START--------

USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS cortex_user_role;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE cortex_user_role;

GRANT ROLE cortex_user_role TO ROLE SYSADMIN;

set c_user = (select CURRENT_USER());

set grant_statement = 'GRANT ROLE cortex_user_role TO USER ' || $c_user || ';';

-- Assign cortex_user_role to current user
EXECUTE IMMEDIATE $grant_statement;

USE ROLE sysadmin;

-- Create demo database
CREATE OR REPLACE DATABASE cortex_analyst_demo;

-- Create schema
CREATE OR REPLACE SCHEMA cortex_analyst_demo.revenue_timeseries;

-- Create warehouse
CREATE OR REPLACE WAREHOUSE cortex_analyst_wh
    WAREHOUSE_SIZE = 'large'
    WAREHOUSE_TYPE = 'standard'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE
COMMENT = 'Warehouse for Cortex Analyst demo';

GRANT USAGE ON WAREHOUSE cortex_analyst_wh TO ROLE cortex_user_role;
GRANT OPERATE ON WAREHOUSE cortex_analyst_wh TO ROLE cortex_user_role;

GRANT OWNERSHIP ON SCHEMA cortex_analyst_demo.revenue_timeseries TO ROLE cortex_user_role;
GRANT OWNERSHIP ON DATABASE cortex_analyst_demo TO ROLE cortex_user_role;

USE ROLE cortex_user_role;

-- Use the created warehouse
USE WAREHOUSE cortex_analyst_wh;

USE DATABASE cortex_analyst_demo;
USE SCHEMA cortex_analyst_demo.revenue_timeseries;

-- Create stage for raw data
CREATE OR REPLACE STAGE raw_data DIRECTORY = (ENABLE = TRUE);

use role accountadmin;
-- Create the integration with Github
CREATE OR REPLACE API INTEGRATION GITHUB_INTEGRATION_FAHAD
    api_provider = git_https_api
    api_allowed_prefixes = ('https://github.com/fahadaz')
    enabled = true
    comment='Fahads repository containing all the awesome code.';

use role cortex_user_role;
-- Create the integration with the Github repository
CREATE GIT REPOSITORY CORTEX_ANALYST_HOL_REPO 
	ORIGIN = 'https://github.com/fahadaz/cortex_analyst_hol' 
	API_INTEGRATION = 'GITHUB_INTEGRATION_FAHAD' 
	COMMENT = 'Fahads repository containing all the awesome code.';

-- Fetch most recent files from Github repository
ALTER GIT REPOSITORY CORTEX_ANALYST_HOL_REPO FETCH;

ls @CORTEX_ANALYST_HOL_REPO/branches/main/data;

-- copy pdfs from github to internal stage
copy files into @raw_data/
from @CORTEX_ANALYST_HOL_REPO/branches/main/data/;

ls @CORTEX_ANALYST_HOL_REPO/branches/main/NOTEBOOK_CORTEX_ANALYST;

-- create notebook stage
create or replace stage notebook_stage DIRECTORY = (ENABLE = TRUE) ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');

-- copy notebook from github to internal stage
copy files into @notebook_stage/
from @CORTEX_ANALYST_HOL_REPO/branches/main/NOTEBOOK_CORTEX_ANALYST;

ls @cortex_analyst_demo.REVENUE_TIMESERIES.NOTEBOOK_STAGE;

-- Create Notebook
CREATE or replace NOTEBOOK __NOTEBOOK_CORTEX_ANALYST_HOL
 FROM '@cortex_analyst_demo.REVENUE_TIMESERIES.NOTEBOOK_STAGE/NOTEBOOK_CORTEX_ANALYST'
 MAIN_FILE = 'notebook_app.ipynb'
 QUERY_WAREHOUSE = 'cortex_analyst_wh';

-- create sis stage
create or replace stage sis_stage DIRECTORY = (ENABLE = TRUE) ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');

-- copy streamlit from github to internal stage
copy files into @sis_stage/
from @CORTEX_ANALYST_HOL_REPO/branches/main/SIS/;

ls @sis_stage;

-- create streamlit page
CREATE OR REPLACE STREAMLIT __CORTEX_ANALYST_HOL
ROOT_LOCATION = '@cortex_analyst_demo.REVENUE_TIMESERIES.sis_stage'
MAIN_FILE = '/cortex_analyst_sis_demo_app.py'
QUERY_WAREHOUSE = 'cortex_analyst_wh';


-------SETUP SCRIPT END--------
```
