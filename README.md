# Snowflake External Function

## Integration with Snowflake

The Snowflake External Function allows you to append Placekeys to your address and POI data stored in Snowflake. The function can be used similarly to a user-defined function in a query.

### Creating the API Integration
- Sign in with an ACCOUNTADMIN role to your Snowflake instance. Alternatively, sign in with a role with the global CREATE INTEGRATION privilege.
- In the query editor, execute the following query:
    ```
    USE ROLE ACCOUNTADMIN;
    ```
    
- Specify the database. For example, if your database is called `DEMO_DB`, run the following:
    ```
    USE DEMO_DB;
    ```

- Create the API Integration:
    ```
    CREATE OR REPLACE API INTEGRATION placekey_api_integration
      API_PROVIDER = aws_api_gateway
      API_AWS_ROLE_ARN = 'arn:aws:iam::886725170148:role/placekey-lambda-production'
      ENABLED = true
      API_ALLOWED_PREFIXES = ('https://lbdl9njufi.execute-api.us-east-1.amazonaws.com/api/')
    ;
    ```
    Here, please be sure to use the `api_aws_role_arn` and `api_allowed_prefixes` in the code block above.

- Create the External function to retrieve Placekeys by using the API Integration created in the above step.
    
    ```
    CREATE OR REPLACE EXTERNAL FUNCTION get_placekeys(
      id number, 
      name varchar, 
      street_address varchar, 
      city varchar, 
      state varchar, 
      postal_code varchar, 
      latitude varchar, 
      longitude varchar, 
      country varchar
    )
      RETURNS variant
      API_INTEGRATION = placekey_api_integration
      HEADERS = ('api-key' = '<PASTE_YOUR_KEY_HERE>')
      MAX_BATCH_ROWS = 1000
      AS 'https://lbdl9njufi.execute-api.us-east-1.amazonaws.com/api/placekeys'
    ;
    ```
    Enter your Placekey API key into `headers = ('api-key': '<PASTE_YOUR_KEY_HERE>')`. If you don't have a Placekey API key, get one for free at [placekey.io](https://dev.placekey.io/default/register).

### Using the External Function Directly

- Create some sample address and point-of-interest data:

    ```
    CREATE OR REPLACE TABLE test_addresses (
      ID VARCHAR(16777216),
      NAME VARCHAR(16777216),
      STREETADDRESS VARCHAR(16777216),
      CITY VARCHAR(16777216),
      STATE VARCHAR(16777216),
      ZIPCODE VARCHAR(16777216),
      LATITUDE VARCHAR(16777216),
      LONGITUDE VARCHAR(16777216),
      COUNTRY VARCHAR(16777216),
      OTHER_COLUMN VARCHAR(16777216)
    );

    INSERT INTO test_addresses
        VALUES 
        ('0', 'Twin Peaks Petroleum', '598 Portola Dr', 'San Francisco', 'CA', '94131', '37.7371', '-122.44283', 'US', 'other_value_1'),
        ('1', null, null, null, null, null, '37.7371', '-122.44283', 'US', 'other_value_2'),
        ('2', 'Beretta', '1199 Valencia St', 'San Francisco', 'CA', '94110', null, null, 'US', 'other_value_3'),
        ('3', 'Tasty Hand Pulled Noodle', '1 Doyers St', 'New York', 'ny', '10013', null, null, 'US', 'other_value_4'),
        ('4', null, '1 Doyers St', 'New York', 'NY', '10013', null, null, null, null);
    ```
    
- To use the external function directly, call it like so:
    
    ```
    SELECT get_placekeys(joined.*) AS result
    FROM (
      SELECT ID, NAME, STREETADDRESS, CITY, STATE, ZIPCODE, LATITUDE, LONGITUDE, COUNTRY
      FROM test_addresses
    ) AS joined;
    ```
    
    Note that the above requires the following:
    - The table needs to have a unique column `id` which is passed as the first argument to the external function. The function returns the id which was originally passed to it along with the Placekeys, so the result can be joined back to the original table. The `id` field cannot be `null`.
    - The order of the fields in the `SELECT` statement should exactly match that of the external function definition: `(id, location_name, street_address, city, region, postal_code, latitude, longitude, country)`.
    - There are at most 1,000 rows in the table to which you want to append Placekeys. To perform queries for >1000 Placekeys, see the procedure below.

    The function can be used as follows to query only a limited number of fields. For example, if test_addresses only had the fields `STREETADDRESS`, `CITY`, and `STATE`, the function could be called as follows:
    
    ```
    SELECT get_placekeys(joined.*) AS result
    FROM (
      SELECT ID, null AS missing_name, STREETADDRESS, CITY, STATE, null AS missing_zip, null AS no_lat, null AS no_lon, null AS defaults_to_us
      FROM test_addresses
    ) AS joined;
    ```
    
### Using the External Function Within a Procedure
    
- To query Placekeys for more than 1,000 rows,  use a precedure. Create a table to map the column names in your table to the Placekey API fields:
    
    ```
    CREATE OR REPLACE TABLE test_lookup (
        STREET_ADDRESS VARCHAR(16777216),
        CITY VARCHAR(16777216),
        REGION VARCHAR(16777216),
        POSTAL_CODE VARCHAR(16777216),
        LOCATION_NAME VARCHAR(16777216),
        LATITUDE VARCHAR(16777216),
        LONGITUDE VARCHAR(16777216),
        ISO_COUNTRY_CODE VARCHAR(16777216)
    );

    insert into test_lookup
        values
        ('STREETADDRESS', 'CITY', 'STATE', 'ZIPCODE', 'NAME', 'LATITUDE', 'LONGITUDE', 'COUNTRY');
    ```
    
    Note that the table `test_lookup` has column headers which match all of the Placekey API fields (`street_address`, `city`, `region`, `postal_code`, `location_name`, `latitude`, `longitude`, and `iso_country_code`) and the first and only row has the names of the fields defined in `test_addresses` above. 
    
    It is not necessary to define all of the Placekey API fields - for example, if the data in your table only contains fields which match `street_address`, `city` `region`, and `postal_code`, you would instead have inserted the following into `test_lookup`:
    
    ```
    INSERT INTO test_lookup
    VALUES
    ('STREETADDRESS', 'CITY', 'STATE', 'ZIPCODE', null, null, null, null);
    ```
    
- Use the following procedure to perform bulk queries:

    ```
    CREATE OR REPLACE PROCEDURE APPEND_PLACEKEYS(
      TBL_QUERY VARCHAR(100), 
      TBL_MAPPING VARCHAR(100),
      TBL_OUT VARCHAR(100),
      TBL_TEMP VARCHAR(100),
      API_FUNCTION VARCHAR(100),
      BATCH_SIZE FLOAT
    )
    RETURNS VARCHAR
    LANGUAGE JAVASCRIPT
    AS $$
        // Copy the input table and add a primary key

        var cmd_seq1 = `CREATE OR REPLACE SEQUENCE seq1 START = 0 INCREMENT = 1;`    
        var stmt_seq1 = snowflake.createStatement( {sqlText: cmd_seq1} );
        var result_seq1 = stmt_seq1.execute();

        var cmd_copytable = `CREATE OR REPLACE TEMPORARY TABLE ${TBL_OUT} LIKE ${TBL_QUERY};`
        var stmt_copytable = snowflake.createStatement( {sqlText: cmd_copytable} );
        var result_copytable = stmt_copytable.execute();

        var cmd_alter = `ALTER TABLE ${TBL_OUT} ADD COLUMN temp_primary_key INT DEFAULT seq1.nextval;`
        var stmt_alter = snowflake.createStatement( {sqlText: cmd_alter} );
        var result_alter = stmt_alter.execute();    

        var cmd_insert = `INSERT INTO ${TBL_OUT} SELECT *, seq1.nextval FROM ${TBL_QUERY};`
        var stmt_insert = snowflake.createStatement( {sqlText: cmd_insert} );
        var result_insert = stmt_insert.execute();

        // Column mapping

        var cmd_map = `SELECT * FROM ${TBL_MAPPING};`
        var stmt_map = snowflake.createStatement( {sqlText: cmd_map} );
        var result_map = stmt_map.execute();
        result_map.next();
        c_location_name = result_map.getColumnValue("LOCATION_NAME");
        c_street_address = result_map.getColumnValue("STREET_ADDRESS");
        c_city = result_map.getColumnValue("CITY");
        c_region = result_map.getColumnValue("REGION");
        c_postal_code = result_map.getColumnValue("POSTAL_CODE");
        c_latitude = result_map.getColumnValue("LATITUDE");
        c_longitude = result_map.getColumnValue("LONGITUDE");
        c_country_code = result_map.getColumnValue("ISO_COUNTRY_CODE");

        // Create a temporary table to store the results of the query

        var cmd_payload = `CREATE OR REPLACE TABLE ${TBL_TEMP} (RESULT ARRAY);`
        var stmt_payload = snowflake.createStatement( {sqlText: cmd_payload} );
        var result_payload = stmt_payload.execute();
        result_payload.next();

        // Query the API

        var cmd_count = `SELECT COUNT(*) FROM ${TBL_OUT};`   
        var stmt_count = snowflake.createStatement( {sqlText: cmd_count} );
        var result_count = stmt_count.execute();
        result_count.next()
        var num_rows = result_count.getColumnValue(1)
        var num_batches = Math.ceil(num_rows / BATCH_SIZE)

        for (var i = 0; i < num_batches; i++) {
          var cmd_api = `
            INSERT INTO ${TBL_TEMP}(RESULT)
              SELECT ${API_FUNCTION}(a.*) AS RESULT FROM (
                SELECT temp_primary_key, 
                ${c_location_name}, 
                ${c_street_address}, 
                ${c_city}, 
                ${c_region}, 
                ${c_postal_code}, 
                ${c_latitude},
                ${c_longitude}, 
                ${c_country_code} 
                FROM ${TBL_OUT} 
                LIMIT ${BATCH_SIZE}
                OFFSET ${BATCH_SIZE * i} ) AS a;
          `;
          var statementLoop = snowflake.createStatement( {sqlText: cmd_api} );
          var result_setLoop = statementLoop.execute();
          result_setLoop.next();
        }

        var cmd_join = `CREATE OR REPLACE TEMPORARY TABLE ${TBL_OUT} AS (
          SELECT p.*, placekey, error
          FROM ${TBL_OUT} p
          INNER JOIN (
            SELECT (seq-1) AS pk1, TRIM(value, '"') AS placekey
            FROM ${TBL_TEMP}, 
            TABLE(FLATTEN(${TBL_TEMP}.result))
            WHERE index = 1
          ) r1 ON r1.pk1 = p.temp_primary_key
          INNER JOIN (
            SELECT (seq-1) AS pk2, TRIM(value, '"') AS error
            FROM ${TBL_TEMP}, 
            TABLE(FLATTEN(${TBL_TEMP}.result))
            WHERE index = 2
          ) r2 ON r2.pk2 = p.temp_primary_key
        )`;
        var stmt_join = snowflake.createStatement( {sqlText: cmd_join} );
        var result_join = stmt_join.execute();

        var cmd_drop = `ALTER TABLE ${TBL_OUT} DROP COLUMN temp_primary_key;`;
        var stmt_drop = snowflake.createStatement({sqlText: cmd_drop});
        var result_drop = stmt_drop.execute();

        return `Done! Data stored in table: ${TBL_OUT}`;
    $$
    ;
    ```
    
    The procedure takes the following input variables: 
    - `TBL_QUERY`: the name of the table in which your address/POI data is stored
    - `TBL_MAPPING`: the name of the column mapping table defined earlier
    - `TBL_OUT`: the name of the table to store the results 
    - `TBL_TEMP`: the name of the temporary table used in the procedure
    - `API_FUNCTION`: the name of the External Function defined earlier
    - `BATCH_SIZE`: number of rows to call in each iteration of the internal loop (maximum 1,000)
    
- The procedure can be called like follows:
    
    ```
    CALL APPEND_PLACEKEYS('test_addresses', 'test_lookup', 'payload', 'temp', 'get_placekeys', 1000);
    ```
    
    The above procedure will store all of the fields in your original table along with two additional columns - placekey and error - in a temporary table called `payload`. There is no requirement that you supply a unique ID since this is handled by the procedure.
    
    
### Summary and Code

To test out this functionality, simply copy and paste the below code block into a blank Snowflake worksheet, replace `<PASTE_YOUR_DB_HERE>` with the database you would like to use, replace `<PASTE_YOUR_KEY_HERE>` with your API key, check the box next to `All Queries` (or select all), and click `Run`.

```
USE ROLE ACCOUNTADMIN;
USE <PASTE_YOUR_DB_HERE>;


CREATE OR REPLACE API INTEGRATION placekey_api_integration
  API_PROVIDER = aws_api_gateway
  API_AWS_ROLE_ARN = 'arn:aws:iam::886725170148:role/placekey-lambda-production'
  ENABLED = true
  API_ALLOWED_PREFIXES = ('https://lbdl9njufi.execute-api.us-east-1.amazonaws.com/api/')
;


CREATE OR REPLACE EXTERNAL FUNCTION get_placekeys(
  id number, 
  name varchar, 
  street_address varchar, 
  city varchar, 
  state varchar, 
  postal_code varchar, 
  latitude varchar, 
  longitude varchar, 
  country varchar
)
  RETURNS variant
  API_INTEGRATION = placekey_api_integration
  HEADERS = ('api-key' = '<PASTE_YOUR_KEY_HERE>')
  MAX_BATCH_ROWS = 1000
  AS 'https://lbdl9njufi.execute-api.us-east-1.amazonaws.com/api/placekeys'
;


CREATE OR REPLACE TABLE test_addresses (
  ID VARCHAR(16777216),
  NAME VARCHAR(16777216),
  STREETADDRESS VARCHAR(16777216),
  CITY VARCHAR(16777216),
  STATE VARCHAR(16777216),
  ZIPCODE VARCHAR(16777216),
  LATITUDE VARCHAR(16777216),
  LONGITUDE VARCHAR(16777216),
  COUNTRY VARCHAR(16777216),
  OTHER_COLUMN VARCHAR(16777216)
);


INSERT INTO test_addresses
    VALUES 
    ('0', 'Twin Peaks Petroleum', '598 Portola Dr', 'San Francisco', 'CA', '94131', '37.7371', '-122.44283', 'US', 'other_value_1'),
    ('1', null, null, null, null, null, '37.7371', '-122.44283', 'US', 'other_value_2'),
    ('2', 'Beretta', '1199 Valencia St', 'San Francisco', 'CA', '94110', null, null, 'US', 'other_value_3'),
    ('3', 'Tasty Hand Pulled Noodle', '1 Doyers St', 'New York', 'ny', '10013', null, null, 'US', 'other_value_4'),
    ('4', null, '1 Doyers St', 'New York', 'NY', '10013', null, null, null, null);


CREATE OR REPLACE TABLE test_lookup (
    STREET_ADDRESS VARCHAR(16777216),
    CITY VARCHAR(16777216),
    REGION VARCHAR(16777216),
    POSTAL_CODE VARCHAR(16777216),
    LOCATION_NAME VARCHAR(16777216),
    LATITUDE VARCHAR(16777216),
    LONGITUDE VARCHAR(16777216),
    ISO_COUNTRY_CODE VARCHAR(16777216)
);


insert into test_lookup
    values
    ('STREETADDRESS', 'CITY', 'STATE', 'ZIPCODE', 'NAME', 'LATITUDE', 'LONGITUDE', 'COUNTRY');


CREATE OR REPLACE PROCEDURE APPEND_PLACEKEYS(
  TBL_QUERY VARCHAR(100), 
  TBL_MAPPING VARCHAR(100),
  TBL_OUT VARCHAR(100),
  TBL_TEMP VARCHAR(100),
  API_FUNCTION VARCHAR(100),
  BATCH_SIZE FLOAT
)
RETURNS VARCHAR
LANGUAGE JAVASCRIPT
AS $$
    // Copy the input table and add a primary key
    
    var cmd_seq1 = `CREATE OR REPLACE SEQUENCE seq1 START = 0 INCREMENT = 1;`    
    var stmt_seq1 = snowflake.createStatement( {sqlText: cmd_seq1} );
    var result_seq1 = stmt_seq1.execute();
  
    var cmd_copytable = `CREATE OR REPLACE TEMPORARY TABLE ${TBL_OUT} LIKE ${TBL_QUERY};`
    var stmt_copytable = snowflake.createStatement( {sqlText: cmd_copytable} );
    var result_copytable = stmt_copytable.execute();
              
    var cmd_alter = `ALTER TABLE ${TBL_OUT} ADD COLUMN temp_primary_key INT DEFAULT seq1.nextval;`
    var stmt_alter = snowflake.createStatement( {sqlText: cmd_alter} );
    var result_alter = stmt_alter.execute();    

    var cmd_insert = `INSERT INTO ${TBL_OUT} SELECT *, seq1.nextval FROM ${TBL_QUERY};`
    var stmt_insert = snowflake.createStatement( {sqlText: cmd_insert} );
    var result_insert = stmt_insert.execute();
        
    // Column mapping
    
    var cmd_map = `SELECT * FROM ${TBL_MAPPING};`
    var stmt_map = snowflake.createStatement( {sqlText: cmd_map} );
    var result_map = stmt_map.execute();
    result_map.next();
    c_location_name = result_map.getColumnValue("LOCATION_NAME");
    c_street_address = result_map.getColumnValue("STREET_ADDRESS");
    c_city = result_map.getColumnValue("CITY");
    c_region = result_map.getColumnValue("REGION");
    c_postal_code = result_map.getColumnValue("POSTAL_CODE");
    c_latitude = result_map.getColumnValue("LATITUDE");
    c_longitude = result_map.getColumnValue("LONGITUDE");
    c_country_code = result_map.getColumnValue("ISO_COUNTRY_CODE");
    
    // Create a temporary table to store the results of the query
    
    var cmd_payload = `CREATE OR REPLACE TABLE ${TBL_TEMP} (RESULT ARRAY);`
    var stmt_payload = snowflake.createStatement( {sqlText: cmd_payload} );
    var result_payload = stmt_payload.execute();
    result_payload.next();
    
    // Query the API
    
    var cmd_count = `SELECT COUNT(*) FROM ${TBL_OUT};`   
    var stmt_count = snowflake.createStatement( {sqlText: cmd_count} );
    var result_count = stmt_count.execute();
    result_count.next()
    var num_rows = result_count.getColumnValue(1)
    var num_batches = Math.ceil(num_rows / BATCH_SIZE)
    
    for (var i = 0; i < num_batches; i++) {
      var cmd_api = `
        INSERT INTO ${TBL_TEMP}(RESULT)
          SELECT ${API_FUNCTION}(a.*) AS RESULT FROM (
            SELECT temp_primary_key, 
            ${c_location_name}, 
            ${c_street_address}, 
            ${c_city}, 
            ${c_region}, 
            ${c_postal_code}, 
            ${c_latitude},
            ${c_longitude}, 
            ${c_country_code} 
            FROM ${TBL_OUT} 
            LIMIT ${BATCH_SIZE}
            OFFSET ${BATCH_SIZE * i} ) AS a;
      `;
      var statementLoop = snowflake.createStatement( {sqlText: cmd_api} );
      var result_setLoop = statementLoop.execute();
      result_setLoop.next();
    }
    
    var cmd_join = `CREATE OR REPLACE TEMPORARY TABLE ${TBL_OUT} AS (
      SELECT p.*, placekey, error
      FROM ${TBL_OUT} p
      INNER JOIN (
        SELECT (seq-1) AS pk1, TRIM(value, '"') AS placekey
        FROM ${TBL_TEMP}, 
        TABLE(FLATTEN(${TBL_TEMP}.result))
        WHERE index = 1
      ) r1 ON r1.pk1 = p.temp_primary_key
      INNER JOIN (
        SELECT (seq-1) AS pk2, TRIM(value, '"') AS error
        FROM ${TBL_TEMP}, 
        TABLE(FLATTEN(${TBL_TEMP}.result))
        WHERE index = 2
      ) r2 ON r2.pk2 = p.temp_primary_key
    )`;
    var stmt_join = snowflake.createStatement( {sqlText: cmd_join} );
    var result_join = stmt_join.execute();
    
    var cmd_drop = `ALTER TABLE ${TBL_OUT} DROP COLUMN temp_primary_key;`;
    var stmt_drop = snowflake.createStatement({sqlText: cmd_drop});
    var result_drop = stmt_drop.execute();
    
    return `Done! Data stored in table: ${TBL_OUT}`;
$$
;

CALL APPEND_PLACEKEYS('test_addresses', 'test_lookup', 'payload', 'temp', 'get_placekeys', 2);

SELECT * FROM payload;
```
