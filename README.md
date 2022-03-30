# #1 Relationships & relationship SOQL queries examples.

There are two main relationship types in Salesforce Lookup Relationship and Master-Detail relationships. There are some differences betweent these relationships:

<center>
    <table>
        <tr>
            <th>Lookup Relationship</th>
            <th>Master-Detail Relationship</th>
        </tr>
        <tr>
            <td>up to 25 per one object</td>
            <td>up to 2 per one object</td>
        </tr>
        <tr>
            <td>parent is not a required field</td>
            <td>parent field on a child is required</td>
        </tr>
        <tr>
            <td>deleting a parent does not delete a child</td>
            <td>deleting a parent automatically deletes a child</td>
        </tr>
        <tr>
            <td>can be multiple layers deep</td>
            <td>a child of one master detail relationship cannot be the parent of another one</td>
        </tr>
        <tr>
            <td>no impact on a security and 
		</td>
            <td>access to a parent determines access to a children</td>
        </tr>
    </table>
</center>

## Lookup relationships:

### Standard objects:

**Account** (**Parent**), **Contact** (**Child**) - Lookup field is on the **Contact** standard object | One **Account** can have 0 or many **Contacts** | One **Contact** can have 0 or one **Account**.

Child-to-parent queries:

```sql
SELECT Name, Phone, Account.Id, Account.Name FROM Contact
```

<p align="center">
    <img src="./screenshots/screenshot-1.png" alt="screen-1.png"/>
</p>

```sql
SELECT Name, Phone, Account.Id, Account.Name FROM Contact WHERE Account.Id = '0010900000iMM6GAAW'
```

<p align="center">
    <img src="./screenshots/./screenshot-2.png" alt="screen-2.png"/>
</p>

Parent-to-child queries:

```sql
SELECT Id, Name, (SELECT Name, Phone FROM Contacts) FROM Account
```

<p align="center">
    <img src="./screenshots/./screenshot-3.png" alt="screen-3.png"/>
</p>

```sql
SELECT Id, Name, (SELECT Name, Phone FROM Contacts) FROM Account WHERE Id = '0010900000iMM6DAAW'
```

<p align="center">
    <img src="./screenshots/./screenshot-4.png" alt="screen-4.png"/>
</p>

### Custom objects:

**Vehicle** (**Parent**), **Driver** (**Child**) - Lookup field is on the **Driver** custom object | One **Vehicle** can have 0 or many **Drivers** | One **Driver** can have 0 or one **Vehicle**

Child-to-parent queries:

```sql
SELECT Name, Nationality__c, Vehicle__r.Id, Vehicle__r.Manufacturer__c FROM Driver__c
```

```sql
SELECT Name, Nationality__c, Vehicle__r.Id, Vehicle__r.Manufacturer__c FROM Driver__c WHERE Vehicle__r.Id = 'a000900000FThKjAAL'

```

Parent-to-child queries:

```sql
SELECT Id, Name, Manufacturer__c, (SELECT Name, Nationality__c FROM Drivers__r) FROM Vehicle__c
```

```sql
SELECT Id, Name, Manufacturer__c, (SELECT Name, Nationality__c FROM Drivers__r) FROM Vehicle__c WHERE Id = 'a000900000FThKjAAL'
```

Notice that it is pretty important to use **__c** suffix when you are using/querying custom object and **__r** suffix when you want to query fields of realated object. Also remember to use plural form in the second SELECT statement when you use parent-to-child queries.

## Master-Detail relationships:

The Master-Detail queries look exactly the same as for the Lookup.

### Standard objects:

**Account** (**Parent/Master**), **Entitlement** (**Child/Detail**) - Master-detail field is on the **Entitlement** standard object | One **Account** can have 0 or many **Entitlements** | One **Entitlement** must have exactly one **Account**.

Child-to-parent queries:

```sql
SELECT Name, Status, Account.Id, Account.Name FROM Entitlement
```
```sql
SELECT Name, Status, Account.Id, Account.Name FROM Entitlement WHERE Account.Id = '0010900000g6FkaAAE'
```

Parent-to-child queries:

```sql
SELECT Id, Name, (SELECT Name, Status FROM Entitlements) FROM Account
```
```sql
SELECT Id, Name, (SELECT Name, Status FROM Entitlements) FROM Account WHERE Id = '0010900000iMM6FAAW'
```

# #2 Batches & scheduled jobs.

The idea of Apex batch jobs is to work on the huge number (thousands / millions) of records. Apex Batch class has to implement Database.Batchable<sObject> interface and its 3 methods:

 - **start** (pre-processing operations) - here the records are being collected
 - **execute** - here some operations are performed on the provided records
 - **finish** (post-processing operations) - this method is called once all operations are done

Here is a pretty simple implementation of this interface. This batch just deletes every Opportunity with the stage name 'Closed Won' or 'Closed Lost'.

```java
public class OpportunitiesCleanerBatch implements Database.Batchable<sObject> {
    public Database.QueryLocator start(Database.BatchableContext context) {
        System.debug('Job started!');
        return Database.getQueryLocator([SELECT StageName FROM Opportunity]);
    }
    
    public void execute(Database.BatchableContext context, List<Opportunity> opportunities) {
        List<Opportunity> opportunitiesToRemove = new List<Opportunity>();
        
        for (Opportunity o : opportunities) {
            if (o.StageName == 'Closed Won' || o.StageName == 'Closed Lost') {
                opportunitiesToRemove.add(o);
            }
        }
        
        delete opportunitiesToRemove;

        System.debug(
            String.format(
                '{0} opportunities with the \"{1}\" or \"{2}\" stage name have been deleted.',
                new List<String> { String.valueOf(opportunitiesToRemove.size()), 'Closed Won', 'Closed Lost' }
            )
        );
    }
    
    public void finish(Database.BatchableContext context) {
        System.debug('Job finished!');
    }
}
```

Here is the code that allows to run the batch above from Developer Console or within another class:

```java
OpportunitiesCleanerBatch oppCleanerBatch = new OpportunitiesCleanerBatch();
Database.executeBatch(oppCleanerBatch);
```

The interface, that works really good with batches is Schedulable interface. It allows to schedule an instance of the Apex class to run at a specific time.

Here is implementation of Schedulable interface that runs OpportunitiesCleanerBatch.

```java
public class OpportunitiesCleanerBatchSchedule implements Schedulable {
    public void execute(SchedulableContext context) {
        OpportunitiesCleanerBatch oppCleanerBatch = new OpportunitiesCleanerBatch();
        Database.executeBatch(oppCleanerBatch);
    }
}
```

There are two possibilities to schedule a job. Obviously you can run the code from the Developer Console or within another class.

```java
OpportunitiesCleanerBatchSchedule scheduledBatch = new OpportunitiesCleanerBatchSchedule();
// String sch = 'SECONDS MINUTES HOURS DAY_OF_MONTH MONTH DAY_OF_WEEK OPTIONAL_YEAR';
String sch = '00 45 6-22 ? * * *';
System.schedule('Opportunities Cleaner Batch', sch, scheduledBatch);
```

You can also use UI to do this: Setup -> Apex Jobs -> Apex Classes -> Schedule Apex. There you have to add job name, select proper class and choose specific time and how often you want the job to run.

# #3 Some useful Apex code snippets.

- Send an email + debug.

```java
String subject = 'Email Subject.';
String body = 'Email Body.';
String[] addresses = new String[] { 'test@mail.com' };

Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();

mail.setToAddresses(addresses);
mail.setSubject(subject);
mail.setPlainTextBody(body);

Messaging.SendEmailResult[] results = Messaging.sendEmail(new Messaging.SingleEmailMessage[] { mail });

for (Messaging.SendEmailResult r : results) {
    System.debug(r);
}
```

- Get picklist values for picklist field (StageName field values of Opportunity standard object in this case).

```java
DescribeFieldResult stageNameField = Opportunity.StageName.getDescribe();

for (Integer i = 0; i < stageNameField.getPicklistValues().size(); i++) {
    System.debug(String.format(' StageName {0}: {1}', new List<String> { String.valueOf(i), stageNameField.getPicklistValues()[i].value }));
}
```

- Run batch manually using Anonymous Apex.

```java
ExampleBatch batch = new ExampleBatch();
Database.executeBatch(batch);
```
- Schedule a job using Anonymous Apex.

```java
ExampleBatchSchedule scheduledBatch = new ExampleBatchSchedule();
// String sch = 'SECONDS MINUTES HOURS DAY_OF_MONTH MONTH DAY_OF_WEEK OPTIONAL_YEAR';
String sch = '00 45 6-22 ? * * *';
System.schedule('Example Batch', sch, scheduledBatch);
```

- Easy way to get set of object Ids.
	
```java
List<SObject> opportunities = [SELECT Id, Name FROM Opportunity LIMIT 10];
Set<Id> opportunitiesIds = (new Map<Id, SObject>(opportunities)).keySet();	
```

# #4 Aura Components + Apex Controllers.

Below you can find an Aura Component which is using Apex Controller to read and display data. It doesn't look so good, but it is one of the simplest examples. In this case data loads after clicking the **Get Opportunities** button.

<p align="center">
    <img src="./screenshots/aura-1.png" alt="aura-1.png"/>
</p>

###### Aura Component: opportunitiesList.cmp

```html
<aura:component implements="flexipage:availableForAllPageTypes" controller="OpportunitiesController">
    <aura:attribute name="opportunities" type="Opportunity[]"/>
    <!-- We are calling component controller method here (NOT Apex method!) -->
    <button onclick="{!c.getOpportunities}">Get Opportunities</button>
    <p>Opportunities list:</p>
    <ul>
        <aura:iteration var="opportunity" items="{!v.opportunities}">
            <li>Id: {!opportunity.Id}, Name: {!opportunity.Name}, StageName: {!opportunity.StageName}</li>
        </aura:iteration>
    </ul>
</aura:component>
```

###### JavaScript Controller: opportunitiesListController.js

```js
({
    getOpportunities: function(component, event, helper){
        // We are calling Apex controller method here.
        var action = component.get("c.getOpportunitiesList");
        action.setCallback(this, function(response){
            var state = response.getState();
            if (state === "SUCCESS") {
                component.set("v.opportunities", response.getReturnValue());
            }
        });
        $A.enqueueAction(action);
    }
})
```

###### Apex Controller: OpportunitiesController.cls

```java
public with sharing class OpportunitiesController {
    @AuraEnabled
    public static List<Opportunity> getOpportunitiesList() {
        return [SELECT Id, Name, StageName, CreatedDate
                FROM Opportunity
                ORDER BY CreatedDate DESC
                LIMIT 10];
    }
}
```

Of course we can use datatable to achieve much better appearance. In this case there is no need to click the button to load data.

<p align="center">
    <img src="./screenshots/aura-2.png" alt="aura-2.png"/>
</p>

###### Aura Component: opportunitiesList.cmp

```html
<aura:component implements="flexipage:availableForAllPageTypes,force:hasRecordId" controller="OpportunitiesController" access="global">
    <aura:attribute name="columns" type="list"/>
    <aura:attribute name="opportunities" type="Opportunity[]"/>
    <!-- We are calling component controller method here (NOT Apex method!) -->
    <aura:handler name='init' action="{!c.getOpportunities}" value="{!this}"/>
    <!-- <button onclick="{!c.getOpportunities}">Get Opportunities</button> -->
    <lightning:card title="Opportunities List">
        <lightning:datatable keyField="id"
                             data="{!v.opportunities}"
                             columns="{!v.columns}"
                             hideCheckboxColumn="true"
                             maxColumnWidth="1000"
                             minColumnWidth="150"/>
    </lightning:card>
</aura:component>
```

###### JavaScript Controller: opportunitiesListController.js

```js
({
    getOpportunities: function (component, event, helper) {
        component.set('v.columns', [
            {label: 'Id', fieldName: 'Id', type: 'text'},
            {label: 'Name', fieldName: 'Name', type: 'text'},
            {label: 'Stage Name', fieldName: 'StageName', type: 'text'},
            {label: 'Creation Date', fieldName: 'CreatedDate', type: 'date'},
        ]);
            
        // We are calling Apex controller method here.
        var action = component.get("c.getOpportunitiesList");
        action.setCallback(this, function(response){
            var state = response.getState();
            if (state === "SUCCESS") {
                component.set("v.opportunities", response.getReturnValue());
            }
        });
        $A.enqueueAction(action);
    }
})
```

# #5 Object Oriented Programming in Apex.

What is Apex?

As the Salesforce documentation says:

> Apex is a strongly typed, object-oriented programming language that allows developers to execute flow and transaction control statements on Salesforce servers in conjunction with calls to the API

It is just an OOP language that is pretty similar to Java or C#.

Below you can find some informations about classes and interfaces:

<center>
    <table>
    <tbody>
        <tr>
            <th></th>
            <th>Virtual</th>
            <th>Abstract</th>
            <th>Interface</th>
        </tr>
        <tr>
            <td>implementation</td>
            <td>full class implementation</td>
            <td>partial class implementation</td>
            <td>it's just a "contract"</td>
        </tr>
        <tr>
            <td>keyword</td>
            <td>extends</td>
            <td>extends</td>
            <td>implements</td>
        </tr>
        <tr>
            <td>can have variables and properties</td>
            <td>yes</td>
            <td>yes</td>
            <td>no</td>
        </tr>
        <tr>
            <td>can have defined methods</td>
            <td>yes</td>
            <td>yes</td>
            <td>no</td>
        </tr>
        <tr>
            <td>can be instantiated (directly)</td>
            <td>yes</td>
            <td>no</td>
            <td>no</td>
        </tr>
        <tr>
            <td>can have abstract methods</td>
            <td>no</td>
            <td>yes</td>
            <td>yes</td>
        </tr>
    </table>
</center>

Here is really simple example, how this all works together in Apex:

###### Tuningable.cls

```java
public interface Tuningable {
    // Interface methods cannot have a body.
	void tuning();
}
```

###### Vehicle.cls

```java
// This class is abstract so it cannot be instantiated. It can be extended.
public abstract class Vehicle {
    // Automatic property - like in C#.
    public String Name { get; set; }
    public String Color { get; set; }
    
    public Vehicle(String name, String color) {
        this.Name = name;
        this.Color = color;
    }
    
	// Abstract method that has to be implemented by the subclass. Abstract methods have no body.
    public abstract Integer getMaxSpeed();
    
    // Virtual method can (but not has to) be overridden by the subclass.
    public virtual String getInfo() {
        return String.format('I am a {0} {1}.', new List<String> { this.Color, this.Name });
    }
}
```

###### BaseCar.cls

```java
// This is a virtual class so it can be extended by other classes. It also can be instantiated.
public virtual class BaseCar extends Vehicle {
    public Integer MaxSpeed { get; set; }
    
    // We can call super class constructor by using 'super' keyword - like in Java.
    public BaseCar(String name, String color, Integer maxSpeed) {
        super(name, color);
        this.MaxSpeed = maxSpeed;
    }
    
    // We can call super class method by using 'super' keyword - like in Java.
    public override Integer getMaxSpeed() {
        return this.MaxSpeed;
    }
    
    // We can call super class method by using 'super' keyword - like in Java.
    // We have to use 'override' keyword if we want to override superclass method.
    public override String getInfo() {
        return String.format('I am a {0} {1}. My max speed is km/h.',
                             new List<String> { super.getInfo(), String.valueOf(this.MaxSpeed) });
    }
}
```

###### SportsCar.cls

```java
// We cannot extend this class. We can extend class only if it is abstract or virtual.
public class SportsCar extends BaseCar implements Tuningable {
    public Boolean HasTurbo { get; set; }
    
    public SportsCar(String name, String color, Integer maxSpeed) {
        super(name, color, maxSpeed);
    }
    
    // We can call another constructor by using 'this' keyword - like in Java.
    public SportsCar(String name, String color, Integer maxSpeed, Boolean hasTurbo) {
        this(name, color, maxSpeed);
        this.hasTurbo = hasTurbo;
    }
    
    // We don't have to (and even cannot) use 'override' keyword if we are implementing interface method.
    public void tuning() {
        maxSpeed += 10;
    }
}
```

# #5 Apex Governor Limits.

What are the Apex Governor Limits?

As the Salesforce documentation says:

> Because Apex runs in a multitenant environment, the Apex runtime engine strictly enforces limits so that runaway Apex code or processes don’t monopolize shared resources.

Below you can find one of the most common limits in Salesforce (the table is not complete):

<center>
<table>
    <tr>
        <th>Description</th>
        <th>Synchronous Limit</th>
        <th>Asynchronous Limit</th>
    </tr>
    <tr>
        <td>number of SOQL queries</td>
        <td>100</td>
        <td>200</td>
    </tr>
    <tr>
        <td>number of records retrieved by SOQL queries</td>
        <td>50,000</td>
        <td>50,000</td>
    </tr>
    <tr>
        <td>number of records retrieved by Database.getQueryLocator</td>
        <td>10,000</td>
        <td>10,000</td>
    </tr>
    <tr>
        <td>number of SOSL queries</td>
        <td>20</td>
        <td>20</td>
    </tr>
    <tr>
        <td>number of records retrieved by a single SOSL query</td>
        <td>2,000</td>
        <td>2,000</td>
    </tr>
    <tr>
        <td>number of DML statements issued</td>
        <td>150</td>
        <td>150</td>
    </tr>
    <tr>
        <td>number of records processed as a result of DML statements</td>
        <td>10,000</td>
        <td>10,000</td>
    </tr>
    <tr>
        <td>heap size</td>
        <td>6 MB</td>
        <td>12 MB</td>
    </tr>
    <tr>
        <td>maximum CPU time on the Salesforce servers</td>
        <td>10,000 ms</td>
        <td>60,000 ms</td>
    </tr>
</table>
</center>
	
# #6 Apex Triggers.

Apex Trigger is a code that executes before or after any operations are performed on the specific record.

Below you can find an example of pretty simple trigger that updates Vehicle name by appending current user name to it.

```java
trigger VehicleTrigger on Vehicle__c (before insert) {
    for (Vehicle__c vehicle : Trigger.New) {
        vehicle.Name += ' of ' +  System.UserInfo.getUserName();
    }
}
```

Possibly types of operations are:

- insert
- update
- delete
- merge
- upsert
- undelete

There are some Trigger context variables that allow you to check which operation fired the trigger or if it was fired before or after the records were saved:

<center>
    <table>
        <tr>
            <th>Variable</th>
            <th>Returns</th>
        </tr>
        <tr>
            <td>isInsert</td>
            <td>true if the trigger was fired due to an insert operation</td>
        </tr>
        <tr>
            <td>isUpdate</td>
            <td>true if the trigger was fired due to an update operation</td>
        </tr>
        <tr>
            <td>isDelete</td>
            <td>true if the trigger was fired due to a delete operation</td>
        </tr>
        <tr>
            <td>isUndelete</td>
            <td>true if the trigger was fired after a record is recovered from the Recycle Bin</td>
        </tr>
        <tr>
            <td>isBefore</td>
            <td>true if the trigger was fired before any record was saved</td>
        </tr>
        <tr>
            <td>isAfter</td>
            <td>true if this trigger was fired after all records were saved</td>
        </tr>
    </table>
</center>

There are also some variables, that allow you to access the records:

<center>
    <table>
        <tr>
            <th>Variable</th>
            <th>Returns</th>
            <th>Records available in</th>
        </tr>
        <tr>
            <td>new</td>
            <td>a list of the new versions of the sObject records</td>
            <td>insert, update, undelete (and can be modified only before)</td>
        </tr>
        <tr>
            <td>newMap</td>
            <td>a map of IDs to the new versions of the sObject records</td>
            <td>before update, after insert, after update, after undelete</td>
        </tr>
        <tr>
            <td>old</td>
            <td>a list of the old versions of the sObject records</td>
            <td>update, delete</td>
        </tr>
        <tr>
            <td>oldMap</td>
            <td>a map of IDs to the old versions of the sObject records</td>
            <td>update and delete</td>
        </tr>
    </table>
</center>

# #7 Queues

// TODO

```java
String userId = UserInfo.getUserId();

// Get Queue using Current User Id.
GroupMember groupMember = [SELECT Id, Group.Name, UserOrGroupId, GroupId
			   FROM GroupMember
			   WHERE UserOrGroupId = :userId
			   LIMIT 1];

// Get all cases where Queue is the owner.
List<Case> cases = [SELECT Id, Status FROM Case WHERE OwnerId = :groupMember.GroupId];

System.debug('Cases for Queue "' + groupMember.Group.Name + '": ' + cases);
```

# #8 Schema class

As the Salesforce documentation says Schema class:
>Contains methods for obtaining schema describe information.

Useful examples:

Get metadata information about custom apps.

```java
Schema.DescribeTabSetResult[] results = Schema.describeTabs();

for (Schema.DescribeTabSetResult result : results) {
    System.debug('Label: ' + result.getLabel());
}
```

```console
03:36:27:072 USER_DEBUG [4]|DEBUG|Label: Sales
03:36:27:072 USER_DEBUG [4]|DEBUG|Label: Service
03:36:27:072 USER_DEBUG [4]|DEBUG|Label: Marketing
```

---

Get sObjects names.

```java
Map<String, Schema.SObjectType> sObjectsMap = Schema.getGlobalDescribe();

for (String key : sObjectsMap.keySet()) {
	Schema.DescribeSObjectResult result = sObjectsMap.get(key).getDescribe();
	System.debug('SObject name: ' + result.getName());
}
```

```console
04:05:09:071 USER_DEBUG [5]|DEBUG|SObject name: OpportunityStage
04:05:09:071 USER_DEBUG [5]|DEBUG|SObject name: LeadStatus
04:05:09:072 USER_DEBUG [5]|DEBUG|SObject name: CaseStatus
```

---

Get picklist entries.

```java
Schema.DescribeFieldResult describeFieldResult = Account.Industry.getDescribe();
List<Schema.PicklistEntry> picklistEntries = describeFieldResult.getPicklistValues();

for (Schema.PicklistEntry picklistEntry : picklistEntries) {
    System.debug('Picklist entry: ' + picklistEntry);
}
```
```console
10:38:41:007 USER_DEBUG [5]|DEBUG|Picklist entry: Schema.PicklistEntry[getLabel=Consulting;getValue=Consulting;isActive=true;isDefaultValue=false;]
10:38:41:007 USER_DEBUG [5]|DEBUG|Picklist entry: Schema.PicklistEntry[getLabel=Education;getValue=Education;isActive=true;isDefaultValue=false;]
10:38:41:007 USER_DEBUG [5]|DEBUG|Picklist entry: Schema.PicklistEntry[getLabel=Electronics;getValue=Electronics;isActive=true;isDefaultValue=false;]
```

---

Get type of sObject.

```java
Schema.DescribeSObjectResult describeResult = Account.sObjectType.getDescribe();
System.debug('SObject type: ' + describeResult.getSObjectType());
```

```console
11:08:36:015 USER_DEBUG [2]|DEBUG|SObject type: Account
```

# #8 Apex data types.

Primitive data types in Apex.

<center>
    <table>
        <tr>
            <th>Data Type</th>
            <th>Description</th>
            <th>Usage example</th>
        </tr>
        <tr>
            <td>Blob</td>
            <td>-</td>
            <td>
            String myString = 'StringToBlob';
            Blob myBlob = Blob.valueof(myString);
            System.assertEquals('StringToBlob', myBlob.toString());
            </td>
        </tr>
        <tr>
            <td>Boolean</td>
            <td>-</td>
            <td>Boolean isValid = true;</td>
        </tr>
        <tr>
            <td>Date</td>
            <td>-</td>
            <td>Date myDate = Date.newInstance(2022, 2, 18); // 2022-02-18 00:00:00</td>
        </tr>
        <tr>
            <td>DateTime</td>
            <td>-</td>
            <td>DateTime myDateTime = DateTime.newInstance(1999, 2, 11, 8, 6, 16); // 2022-02-18 13:29:15</td>
        </tr>
        <tr>
            <td>Decimal</td>
            <td>-</td>
            <td>Decimal phi = 1.618033;</td>
        </tr>
        <tr>
            <td>Double</td>
            <td>-</td>
            <td>Double pi = 3.14159;</td>
        </tr>
        <tr>
            <td>Id</td>
            <td>-</td>
            <td>0017Q000008Yo6JQAS</td>
        </tr>
        <tr>
            <td>Integer</td>
            <td>-</td>
            <td>Integer count = 15;</td>
        </tr>
        <tr>
            <td>Long</td>
            <td>-</td>
            <td>Long amount = 1337;</td>
        </tr>
        <tr>
            <td>Object</td>
            <td>-</td>
            <td>Object color = 'Red';</td>
        </tr>
        <tr>
            <td>String</td>
            <td>-</td>
            <td>String name = 'Adrian';</td>
        </tr>
        <tr>
            <td>Time</td>
            <td>-</td>
            <td>Time myTime = Time.newInstance(13, 47, 35, 570); // 13:47:35.570Z</td>
        </tr>
    </table>
</center>