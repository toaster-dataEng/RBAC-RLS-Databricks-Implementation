# RBAC-RLS-Databricks-Implementation
RLS (Row-level Security) Implementation on Unity Catalog initiated Databricks. Using ROW FILTER

1.	USE CASE

Members of each security group will go through as reader to an identified Secured Table which will have the functionality to filter rows according to Department and Functional Area they belong to.

	Define Security Groups:
o	Ensure your security groups are defined or provisioned via Entra ID / AAD (Active Directory), and users are assigned to these groups. Members User Email, Security Group Name and Department Code or Id, Functional Area are the critical fields that a security table should have.
o	There are 2 security tables maintained which will be the source of truth for security table and its associated member’s email, department code and shortname, functional area. [sec_group_dept_assignment] and [sec_group_email_distrolist] are the initial tables we utilized for testing until the official security groups were provisioned and now maintained by LAC which are [sec_group_map] and [sec_user_group] 

	Create a Row-Level Security Function:
o	Create a function that filters rows based on the user’s security group. This function will be used to enforce RLS.
o	Create new table based on MV (materialized view) data. As of this writing, ROW FILTER is only limited to table, hence, the approach.
o	Apply ROW FILTER to identified tables that will have the RLS.


2.	PREREQUISITES

	Databricks Administrator (or necessary permissions) – for ROW FILTER implementation (create table, create view, create function, alter table, update table)
	Table Owner – for GRANT access
	Security Dataset – Comprises of essential fields such as Security Group Name, Members Email, User_id, Department Code, Functional Area and other fields that will make the rows unique.
	RBAC Matrix Document – Tables impacted for RLS implementation. Security Group Mapping matrix (3-level namespace: Catalog, Schema, Table/View). 
	Tags should provisioned and serve as identifier for each affected table from which Functional Area it belongs to.

RLS Setup

![image](https://github.com/user-attachments/assets/e745f3e4-90e8-4645-af9f-d977c4b592cf)
![image](https://github.com/user-attachments/assets/b9390017-47aa-420d-ad31-c92ac92a3329)

Additional Reminders:
	Security roles should be in the Entra ID / AAD.
	Users / Members of these security roles / groups should be reflected in Entra ID / AAD.
	In Databricks, SCIM should be provisioned to be able to setup and sync these security groups into the catalog. 
	Security tables (security.entra_id.sec_user_group and security.entra_id.sec_group_map) should be maintained regularly to ensure proper mapping of users, groups, departments they are assigned and functional area. This is a critical reference for RLS (row-level security) of departmental roles.

Maintenance and Support: 
	Security tables should be maintained regularly. It is expected that whenever a user / group is added in Entra Id / Active Directory, these changes should be reflected / synced in the security tables using SCIM provisioning.
	Ensure to recreate the table to make sure that the data are updated everytime the materialized views were refreshed as needed or as scheduled. 

Setup

1.	Identify target tables planned to be secured. Which catalog and schema it belongs to. 
2.	Note that if the table or object subject to be secured is a physical table then we can proceed on the 3rd step but if it’s otherwise, or in this case a materialized view, then we need to convert and load the data into a new table first before we can apply the ROW FILTER. This is the current limitation on implementing ROW FILTER per Microsoft.
3.	Convert MV into a new table. Data is loaded from the MV into the new table. 

CREATE TABLE catalog.schema.newtable AS SELECT * FROM catalog.schema.materialized_view;

4.	Apply or implement the row filter function with the parameter to be used as the reference of the RLS within the table. In this case, the department_code and the table tag which will serve as the functional_area.
5.	Create UDF (User Defined Function) that will served as the RLS function (use case: If user is central role, then all rows otherwise departmental roles with specific department codes mapped.

![image](https://github.com/user-attachments/assets/4264a475-23f3-4b7e-90e0-d566a58b5261)
   
6. Apply ROW FILTER in the new table with the parameters department_code and table name (sample: ‘sample_mv_tbl’). The table name will serve as the identifier from which the RLS is currently applied while the table is accessed by the current user.
   
ALTER TABLE catalog.schema.newtable SET ROW FILTER catalog.schema.newUDF ON (department_code, 'newtable');

7. Setup Central Roles for each new table


reference:
Row and Column Level Security with Unity Catalog | Databricks - https://www.databricks.com/resources/demos/tours/governance-uc/row-and-column-level-security-with-unity-catalog
