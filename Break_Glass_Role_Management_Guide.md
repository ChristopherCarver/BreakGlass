# Break Glass Role Management (-=DRAFT=-) (-=IN PROGRESS=-)

---
[[TOC]]

---
# About
\
_**Break glass role management**_ (BGRM) is a role management framework where accounts are only provided role rights when needed. Unlike persistent roles where the rights granted remain available at all times, break glass roles are only granted for a limited time.  

This guide is to help ServiceNow admins setup BGRM for user roles insuring accounts are properly setup for privileged rights. The goal for BGRM is to work in conjunction with the default out-of-the-box (OOTB) ServiceNow role base security model to insure future upgradeability across ServiceNow releases. Therefore any role based security already setup in ServiceNow will not be impacted.

---
### How BGRM works
\
BGRM uses the existing OOTB roles and access control lists used in ServiceNow. Refer to the ServiceNow documentation for information on [roles](https://docs.servicenow.com/csh?topicname=c_Roles&version=page) and [access control lists](https://docs.servicenow.com/en-US/bundle/tokyo-platform-administration/page/administer/contextual-security/concept/access-control-rules.html)). The only difference with BGRM is you do not grant an account the role directly like you do with ServiceNow, you utilize the BGRM framework to automatically grant and revoke the role from the account.

Once the break glass role(s) and ACL(s) are created (just as you would normally do standard ServiceNow roles and ACLs), you will add the role to the [u_break_glass_role] table. This table contains all of the break glass roles that can be checked out; along with meta data pertaining to the behavior of the role. 

There will be a catalog item called **Break Glass Account Management** that will allow accounts to request access to the roles listed in the [break_glass_role] table. All requests will have to be approved by the support group associated with the role that is defined on the [u_break_glass_role] table. Once approved, the account will be added to the [u_break_glass_user_role] table and the role **break_glass_user** will added to the requested for account. The table [u_break_glass_user_role] contains all of the accounts that have access to specific break glass roles.

Accounts with the **break_glass_user** role will have access to the catalog item **Break Glass Role Management**, where the account can request the temporary provisioning of a role for only their account. Accounts without the **break_glass_user** role will not have access to the catalog item **Break Glass Role Management**. The flow for the request will determine if it can automatically grant the role to the account and then forcibly terminate all of the account's sessions; hence requiring a new login. This forces ServiceNow to re-evaluate the account and granting them the proper rights and privileges of the new role.

In the background, once an account's role is expired, the role will be automatically removed from the account and the account's sessions will be forcefully terminated.

!!! note MFA with BGRM
    ServiceNow offers multi-factor authentication (MFA) with one-time password (OTP) for role based accounts. The BGRM is built around just in time authorization for security, so it would make sense to consider leveraging MFA OTP on the roles created for BGRM. That way, when a role is granted to an account and the account is forced to re-login, then the account will be presented with a MFA OTP prompt for additional security. Refer to the ServiceNow documentation for setting up [Multi-factor authentication](https://docs.servicenow.com/csh?topicname=c_MultifactorAuthentication&version=page).

---
### BGRM versus Elevated Roles
\
ServiceNow does not have a break glass framework or model for roles. While it is possible to designate a role as an elevated role, role elevation restriction only remains on the platform navigation of ServiceNow, not on the portals or applications. Also, role elevation lacks some fundamental functions that is needed for BGRM.

The BGRM framework has the following features:

- Time based expiration
- Checkout cap

Each feature is documented in their section below.

---
## Follow along
\
ServiceNow offers many ways to solve a problem or configure an operational business model. These instructions are _a_ way and does not represent _the_ way on creating a break glass role framework. The best way to use these instructions is to read through them and see how the framework can be adopted and modified to fit your organization requirements. 

!!! note Update sets 
    There is an update set that implement this guide; _Break Glass Role Management_. The update set can be found at [https://github.com/ChristopherCarver/BreakGlassRoleMgt](https://github.com/ChristopherCarver/BreakGlassRoleMgt).   

!!! danger Modifying OOTB Tables
    This guide focuses on creating a robust framework tied closely to the out of the box (OOTB) tables already existing in ServiceNow. There are alternative implementation solutions within ServiceNow to accomplish the same result. Your mileage may vary depending on scope defined by and practices set by your organization and/or development team. This framework is meant to be as open and flexible to meet varying modifications to suit present and future requirements.

!!! note Cost Impact
    Custom tables are created in this framework and there could be a financial impact creating custom tables. ServiceNow allots a set amount of custom tables within an instance that a customer can create free of charge. After the set amount of custom tables are created, ServiceNow charges for custom tables. Talk with your ServiceNow Enterprise Account Executive to learn more.

!!! note Reduce Cost Impact
    To reduce any cost impact to your organization, you can extend OOTB tables free of charge. This guide leaves it to the implementation team to determine the best course of action.

---
# BGRM Foundation
\
The foundation for the BGRM framework is based on the premise that the framework is separate from the default OOTB ServiceNow security model. The only overlap between the BGRM framework and standard ServiceNow user role model is the BGRM relies on ServiceNow to enforce role based security once an account has been granted a role by the BGRM framework. The BGRM framework takes care of the management of the roles independently.   

---
## Add break_glass_user account role
\
Accounts with the **break_glass_user** role will have access to the catalog item **Break Glass Role**.

_Instructions:_

1. Navigate to **User Administration** > **Roles**, click **New**.
1. Under *Role New Record*, fill in the following fields:
    - _Name:_ break_glass_user
    - _Description:_ Account can request additional roles.
1. Click **Submit**, which will create the new role.

---
## Break glass role
\
The _Break Glass Role_ [u_break_glass_role] table contains all of the break glass roles available. This table will need to be manually populated by the admin. 

The _Break Glass Role_ [u_break_glass_role] table will contain the following custom fields:
|Field|Description|
|-----|-----------|
|Name|The name of the break glass role.|
|Description|A description of the break glass role.|
|Role|Reference to the ServiceNow OOTB table _Role_ [sys_user_role].|
|User Criteria|Reference to the ServiceNow OOTB table _User Criteria_ [user_criteria].|
|Lifespan|The number of minutes an account can be granted the role; default 4 hours.|
|Expiring|When the break glass role is 90% near expired. This is calculated based on _Lifespan_.|
|Expires|When the break glass role expires. This is calculated based on _Lifespan_.|

_Instructions:_

1. Navigate to **System Definition** > **Tables**.
1. Click **New**.
1. Under the **Table New record** section, fill in the following fields:
    - _Label:_ Break Glass Role 
    - _Name:_ u_break_glass_role
    - _Add module to menu:_ User Administration
1. In the record header, right-click and select **Save**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Name
    - _Column name:_ (this should default to u_name)
    - _Max length:_ 16
    - _Mandatory:_ true
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ String
    - _Column label:_ Description
    - _Column name:_ (this should default to u_description)
    - _Max length:_ 64
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Reference
    - _Column label:_ Role
    - _Column name:_ (this should default to u_role)
    - _Mandatory:_ true
1. In the **Reference Specification** tab, fill in the following field:
    - _Reference:_ Role [sys_user_role]
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Reference
    - _Column label:_ User Criteria
    - _Column name:_ (this should default to u_user_criteria)
    - _Mandatory:_ true
1. In the **Reference Specification** tab, fill in the following field:
    - _Reference:_ User Criteria [user_criteria]
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Integer
    - _Column label:_ Lifespan
    - _Column name:_ (this should default to u_lifespan)
1. In the record header, right-click and select **Save**.
1. In the **Default Value** tab, fill in the following field:
    - _Default Value:_ 240
1. Click **Update**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ True/False
    - _Column label:_ Expiring
    - _Column name:_ (this should default to u_expiring)
1. Under **Related Links**, click **Advanced view**.
1. In the **Calculated Value** tab, fill in the following fields:
    - _Calculated:_ true
    - _Calculation Type:_ Script
    - _Calculation:_
        ```
        (function calculatedFieldValue(current) {

            var lifespan = current.getValue('u_lifespan'); // minutes
            var lifespanExpiring = parseInt(lifespan * 0.9, 10); // 90% of the lifespan in minutes
        
            var now = new GlideDateTime();
            now.addSeconds(lifespanExpiring * 60);
            var expiring = now.getDisplayValue();
            
            var date = expiring.toString().split(" ")[0];
            var time = expiring.toString().split(" ")[1];
            var hours = time.split(":")[0];
            var minutes = time.split(":")[1];
            var finalExpiring = date + " " + hours + ":" + minutes + ":00";
            
            current.setValue('u_expiring', finalExpiring);
            return finalExpiring; // return the calculated value
        
        })(current);
        ```
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ True/False
    - _Column label:_ Expires
    - _Column name:_ (this should default to u_expires)
1. Under **Related Links**, click **Advanced view**.
1. In the **Calculated Value** tab, fill in the following fields:
    - _Calculated:_ true
    - _Calculation Type:_ Script
    - _Calculation:_
        ```
        (function calculatedFieldValue(current) {

            var lifespan = current.getValue('u_lifespan'); // minutes
            
            var now = new GlideDateTime();
            now.addSeconds(lifespan * 60);
            var expires = now.getDisplayValue();
            
            var date = expires.toString().split(" ")[0];
            var time = expires.toString().split(" ")[1];
            var hours = time.split(":")[0];
            var minutes = time.split(":")[1];
            var seconds = time.split(":")[2];
            if(seconds >= 30) {
                minutes += 1;
            }
            var finalExpires = date + " " + hours + ":" + minutes + ":00";
            
            current.setValue('u_expires', finalExpires);
            return finalExpires;
        
        })(current);
        ```
1. Click **Submit**.
1. Click **Update**.

---
## Break glass role slush bucket
\
The _Break Glass Role_ [u_break_glass_role] table is referenced by the _Break Glass Roles_ catalog item and the reference fields need to defined in a slush bucket for selection. 

1. In the browser URL type in **https://_{your instance}_/u_break_glass_role_list.do?sysparm_view=sys_ref_list**, where {your instance} is the name of your ServiceNow instance.
1. Right-click in the table column header and select **Configure** > **List Layout**. 
1. Under the **Selected** list, remove the **Sys ID** column; click **Sys ID** and then **<** to remove from the **Selected** list.
1. In under the **Available** list add the following columns to the **Selected** list by selecting the name of the column and click **>**.
    - Name
    - Description
1. Click **Save**.

---
## Break glass user role
\
The _Break Glass User Role_ [u_break_glass_user_has_role] table contains the relationship between the account and the break glass role. This table should be maintained by the catalog item _Break Glass Account Management_ and the supporting flow. 

The _Break Glass User Role_ [u_break_glass_user_has_role] table will contain the following custom fields:
|Field|Description|
|-----|-----------|
|User|Reference to the ServiceNow OOTB table _User_ [sys_user].|
|Role|Reference to the custom table _Break Glass Role_ [u_break_glass_role].|
|Active|Designation that the account has been granted the role.|
|Expiration|When the role is set to expire on the account.|
|Expired|Answers if the break glass role has expired base on _Expiration_. This is calculated based on _Expiration_.|
|Request item|Reference to the ServiceNow OOTB table _Requested Item_ [sc_req_item].|

_Instructions:_

1. Navigate to **System Definition** > **Tables**.
1. Click **New**.
1. Under the **Table New record** section, fill in the following fields:
    - _Label:_ Break Glass User Role 
    - _Name:_ u_break_glass_user_role
    - _Add module to menu:_ User Administration
1. In the record header, right-click and select **Save**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Reference
    - _Column label:_ User
    - _Column name:_ (this should default to u_user)
    - _Mandatory:_ true
1. In the **Reference Specification** tab, fill in the following field:
    - _Reference:_ User [sys_user]
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Reference
    - _Column label:_ Break Glass Role
    - _Column name:_ (this should default to u_break_glass_role)
    - _Mandatory:_ true
1. In the **Reference Specification** tab, fill in the following field:
    - _Reference:_ Role [u_break_glass_role]
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ True/False
    - _Column label:_ Active
    - _Column name:_ (this should default to u_active)
1. In the **Default Value** tab, fill in the following field:
    - _Default Value:_ true
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Date/Time
    - _Column label:_ Expiration
    - _Column name:_ (this should default to u_expiration)
    - _Mandatory:_ true
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ True/False
    - _Column label:_ Expired
    - _Column name:_ (this should default to u_expired)
1. Under **Related Links**, click **Advanced view**.
1. In the **Calculated Value** tab, fill in the following fields:
    - _Calculated:_ true
    - _Calculation Type:_ Script
    - _Calculation:_
        ```
        (function calculatedFieldValue(current) {

            // If there is no expiration set, return false.
            if (current.u_expiration.nil()) {
                current.setValue('u_expired',false);
                return false;
            }
        
            var dateToday = new GlideDateTime();
            // If the expiration is less than today's date and time, return false.
            if (current.u_expiration > dateToday) {
                current.setValue('u_expired',false);
                return false;
            }
        
            // Default to true
            current.setValue('u_expired',true);
            return true; // return the calculated value
        
        })(current);
        ```
1. Click **Submit**.
1. In the **Columns** table, click **New**.
1. In the **Dictionary Entry New record** section, fill in the following fields:
    - _Type:_ Reference
    - _Column label:_ Request item
    - _Column name:_ (this should default to u_request_item)
    - _Mandatory:_ true
1. In the **Reference Specification** tab, fill in the following field:
    - _Reference:_ Requested Item [sc_req_item].|
1. Click **Submit**.
1. Click **Update**.

---
## Break Glass Util
\
The _BreakGlassUtil_ script include supports various break glass operations used in break glass catalog items and flows.

_Instructions:_

1. Navigate to **System Definition** > **Script Includes**.
1. Click **New**.
1. In the **Script Include New record** section, fill in the following fields:
    - _Name:_ BreakGlassUtil
    - _API Name:_ (this should default to global.BreakGlassUtil)
    - _Client callable:_ false
    - _Description:_ Utility operations in the support for break glass roles. See [u_break_glass_role] and [u_break_glass_user_has_role] tables. 
    - _Script:_
        ```
        var BreakGlassUtil = Class.create();
        BreakGlassUtil.prototype = {
            initialize: function() {},

            // returns a comma delimited string of break glass roles the user has access to
            getRoles: function() {
                var breakGlassRoleList = [];

                // get all the break glass roles
                var breakGlassRole = new GlideRecord('u_break_glass_role');
                breakGlassRole.query();
                while (breakGlassRole.next()) {
                    var criteria = [];
                    criteria.push(breakGlassRole.getValue('u_user_criteria'));
                    // if the user matches the user criteria for the role
                    if (sn_uc.UserCriteriaLoader.userMatches(gs.getUserID(), criteria)) {
                        // then store it in the array
                        breakGlassRoleList.push(breakGlassRole.getValue('sys_id'));
                    }
                }

                // return the list of break glass roles the user has access to as a comma delimited         string
                return breakGlassRoleList.join(',');
            },
            type: 'BreakGlassUtil'
        };
        ```
1. Click **Submit**.

---
## Break Glass Roles
\
The _Break Glass Roles_ catalog item is a self-service account elevation role provisioning service.

_Instructions:_

1. Navigate to **Service Catalog** > **Catalog Definitions** > **Maintain Items**.
1. Click **New**.
1. In the **Catalog item New record** section, fill in the following fields:
    - _Name:_ Break Glass Roles
    - _Catalogs:_ Service Catalog (*choose the right catalog for you*)
    - _Category:_ Role Delegation (*choose the right category for you*)
    - _Fulfillment automation level:_ Fully automated
1. In the record header, right-click and select **Save**.
1. In the **Item Details** tab, fill in the following fields:
    - _Short Description:_ Temporary account role elevation.
    - _Description:_

        **About:**
        The _Break Glass Roles_ catalog item temporary grants your account an elevated break glass roles. A break glass role is a temporary privileged role that an account checks out to perform elevated operations.

        **Usage:**
        Only the break glass roles that are available to your account will appear in the **Role** field. 

        **Important:**
        - This is a fully automated request. After your submit your request, your ServiceNow session will be forcible terminated by ServiceNow and you will have to log back into ServiceNow. Upon logging into ServiceNow your account will have the requested role temporarily.
        - Once your temporary break glass role expires, the role will automatically be removed from your account and your ServiceNow session will be forcible terminated by ServiceNow.
        - If your ServiceNow session is not terminated, then check your request to determine the reason why your request was not granted. 

1. In the record header, right-click and select **Save**.
1. In the **Variables** tab, click **New**.
1. In the **Variable New record** section, fill in the following fields:
    - _Type:_ List Collector
    - _Mandatory:_ true
    - _Order:_ 100
1. In the **Question** tab, fill in the following fields:
    - _Question:_ Roles
    - _Name:_ (should default to roles)
1. In the **Type Specifications** tab, fill in the following fields:
    - _List table:_ Break Glass Role [u_break_glass_role]
    - _Reference qualifier:_
        ```
        javascript: var query;
        query = "sys_idIN" + new BreakGlassUtil().getRoles();
        query;
        ```
    - _Variable attributes:_ 
        ```
        no_filter,ref_auto_completer=AJAXTableCompleter,ref_ac_columns=u_name;u_description,ref_ac_order_by=u_name
        ```
1. Click **Update**.
1. In the **Variables** tab, click **New**.
1. In the **Variable New record** section, fill in the following fields:
    - _Type:_ Single Line Text
    - _Mandatory:_ true
    - _Order:_ 200
1. In the **Question** tab, fill in the following fields:
    - _Question:_ Justification
    - _Name:_ (should default to justification)
1. Click **Submit**.
1. Click **Update**.

---
## Break Glass Roles Flow
\
The _Break Glass Roles Flow_ is a flow designer process engine for the _Break Glass Roles_ catalog item. 

_Instructions:_

1. Navigate to **Process Automation** > **Flow Designer**. A new tab will open for _Flow Designer_.
1. In **Flow Designer** click **New** and select **Flow**.
1. In the **Flow properties** window, fill in the following fields:
    - _Flow name:_ CAT Break Glass Roles (*CAT represents this flow is associated with a catalog item*)
    - _Description:_ Processes Break Glass Role service requests.
    - _Application:_ Global
    - _Run As:_ System User
1. Click **Submit**.
1. Click **More Actions menu** (the ... on top right) and select **Flow Variables**.
1. Click **Add new input** (the + ) and fill in the following fields:
    - _Label:_ Granted Role
    - _Name:_ granted_role
    - _Type:_ True/False
1. Close the **Flow Variables** window. 
1. Under **Trigger**, click **Add a trigger** and under **Application** select **Service Catalog**.
1. Click **Done**.
1. The following instructions are adding operations under the **Actions** section. Due to the difficulty of documenting flow operations, the folloing sub-ordered steps represent the order of flow operations. At the start of each action there will either be the choice **Add an Action, Flow Logic, or Subflow** or a grouping of **Action**, **Flow Logic**, and **Subflow**. The step will assume general knowledge of which to select and finalizing the operation by clicking **Done**. 
    1. **Flow Logic** > **Set Flow Variables**, fill in the following variables:
        - _Name:_ Granted Role
        - _Data:_ false
    1. **Action** > **ServiceNow Core** > **Get Catalog Variables**, fill in the following fields:
        - _Submitted Request [Requested Item]:_ From the  **Data Pill Picker**, select **Trigger - Service Catalog** > **Requested Item Record**.*
        - _Template Catalog Items and Variables Sets [Catalog Items and Variable Sets]:_ Break Glass Roles
        - _Catalog Variables:_ *Under **Available** list click **roles** and then **>** to move to **Selected**.*
    1. 



---
# Outro

You have now successfully setup a break glass role management framework. Congratulations.



-=NOTES=-
- SNC.UserCriteriaLoader.getAllUserCriteria() returns a list of all the sysid of user criteria the user meets