# Creating the Machine Learning Model

## Introduction

In this lab, you will:

- In ADW:
  - Log into SQL Developer Web and enable user ml_user access to sqldeveloper web.
  - Grant storage priviledges to ml_user
  - Log in with ml_user and export the model to a temporary table.
- In ATP:
  - Create a database link that will be used to copy (pull) an export of the ml model from ADW to ATP.
  - Create a new ml_user and grant that user access to sqldeveloper web, and to storage.
  - Copy the model from ADW to ATP.
  - Log in with ml_user and import the ml model.
  - Create a virtual column on the table that applies the model to rows in the table.

## **Step 1:** Sign in to Oracle Cloud

- Log in, if you have not already done so.

![](./images/2/002.png  " ")

![](./images/2/003.png  " ")

![](./images/2/009.png  " ")

![](./images/2/004.png  " ")


## **Step 2:** Navigate to SQL Developer Web

- Next we will log into SQL Developer Web. Start by navigating to the ADW Instance page and clicking on `Service Console`

  ![](./images/1/0311.png  " ")

- Click on `Development`

  ![](./images/1/0023.png  " ")

- Select `SQL Developer Web`

  ![](./images/1/0026.png  " ")

- Log in with `Username` `admin` and `Password` as the password you used when you created the ADW Instance back in Lab 2

  ![](./images/1/0025.png  " ")

## **Step 3:** Connection ADW ml_user

- Here we will list the mining model that as created in the notebook

- Signed into SQL Developer Web with the **ml_user credentials**, copy and paste the following:

```
<copy>select * from user_Mining_models;</copy>
N1_CLASS_MODEL    CLASSIFICATION    DECISION_TREE    NATIVE    05-MAR-20        0    NO    

```
- In lab 1, we downloaded the ADW wallet zip file, uploaded it to object storage, and created an auth\_token. Copy the wallet downloaded from Lab 1 to the directory data\_pump\_dir by entering the following into SQL Developer Web. Note: make sure to **change the object_uri and tenancy to reflect your own data center** (eg. phoenix and orasenatdpIntitegration02).

```
<copy>BEGIN
     DBMS_CLOUD.GET_OBJECT(
        credential_name => 'api_token',
        object_uri => 'https://objectstorage.us-phoenix-1.oraclecloud.com/n/orasenatdpltintegration02/b/dgcadw/o/cwallet.sso',
        directory_name => 'DATA_PUMP_DIR');
END;
/</copy>
```

- List the files int he data\_pump\_dir by entering the following:

```
<copy>SELECT * FROM DBMS_CLOUD.LIST_FILES('data_pump_dir');</copy>
cwallet.sso    6733        17-MAR-20 04.36.25.000000000 PM GMT    17-MAR-20 04.36.25.000000000 PM GMT
```

## **Step 4:** Create Connection to ADW admin user

- Log out of SQL Developer Web, and then re-log back in as **admin** uesr and paste the following. Note, use the password that you used when you created the ADW.

```
<copy>
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'adw_db',
    username => 'ADMIN',
    password => '&lt;password&gt;'
  );
END;
/</copy>
```

- Next, we want to create a database link. We will need the following information: the hostname, port, service\_name, ssl\_server\_cert_dn. We can obtain this information from the tnsnames.ora file in the ADW wallet.

- Navigate to your ADW wallet file and open the tnsnames.ora file.

![](./images/3/001.png  " ")

- Copy  the port, hostname, service\_name, ssl\_server\_cert_dn, you will need it in the next step.

![](./images/3/002.png  " ")

- Copy the following code snippet, replacing the **hostname**, **port**, **service\_name**, **ssl\_server\_cert_dn** with the values you just copied from the tnsnames.ora file.

```
<copy>
begin
dbms_cloud_admin.drop_database_link('atplink');
end;
/

BEGIN
     DBMS_CLOUD_ADMIN.CREATE_DATABASE_LINK(
          db_link_name => 'atplink',
          hostname => '&gt;hostname from tnsnames.ora&gt;',
          port => '1522',
          service_name => '&lt;service name from tnsnames.ora&gt;',
          ssl_server_cert_dn => '&lt;ssl_server_cert from tnsnames.ora&gt;',
          credential_name => 'atp_db',
          directory_name => 'DATA_PUMP_DIR');
END;
/

select sysdate from dual@adwlink;
/</copy>
```

- Log out of admin user and log back in as **ml_user** and past the following into SQL Developer Web:

```
<copy>
create table temp(my_model blob);

DECLARE
 v_blob blob;
BEGIN
 dbms_lob.createtemporary(v_blob, FALSE);
 dbms_data_mining.export_sermodel(v_blob, 'N1_CLASS_MODEL');
 insert into temp values(v_blob);
 commit;
 dbms_lob.freetemporary(v_blob);
END;
/

select length(my_model) from temp;</copy>
85398
```

## **Step 5:** Create Connection to ATP admin user

- Create a new ml_user, and add additional permissions.
```
<copy>
create user ml_user identified by WelCome1234_;
grant dwrole, oml_developer to ml_user;
create table temp(my_model blob);
grant all on temp to ml_user;
ALTER USER ml_user QUOTA 100M ON DATA;
</copy>
```

- As the **ADW admin user** Copy the model from ADW to ATP into a temp table
```
<copy>
insert into temp@atplink select * from ml_user.temp;
commit;
</copy>
```

- As the **ATP ml_user user** Import the model:
```
<copy>
DECLARE
 v_blob blob;
BEGIN
 dbms_lob.createtemporary(v_blob, FALSE);
  select my_model into v_blob from admin.temp;
 dbms_data_mining.import_sermodel(v_blob, 'N1_CLASS_MODEL');
 dbms_lob.freetemporary(v_blob);
END;
/

select * from user_Mining_models;
</copy>
N1_CLASS_MODEL    CLASSIFICATION    DECISION_TREE    NATIVE    17-MAR-20        0    NO    
```

- As the **ATP ml_user user**, test the model.
```
<copy>
select prediction_probability(N1_CLASS_MODEL, 'Good Credit'
  USING 'Rich' as WEALTH, 2000 as income, 'Silver' as customer_value_segment) Prediction_Probability
from dual;
</copy>
0.506275824432837
```
