# # 1 Relationships & relationship SOQL queries examples

There are two main relationship types in Salesforce Lookup Relationship and Master-Detail relationships. There are some differences betweent these relationships:

<center>
    <table>
        <tr>
            <th>
                Lookup Relationship
            </th>
            <th>
                Master-Detail Relationship
            </th>
        </tr>
        <tr>
            <td>
                up to 25 per one object
            </td>
            <td>
                up to 2 per one object
            </td>
        </tr>
        <tr>
            <td>
                parent is not a required field
            </td>
            <td>
                parent field on a child is required
            </td>
        </tr>
        <tr>
            <td>
                deleting a parent does not delete a child
            </td>
            <td>
                deleting a parent automatically deletes a child
            </td>
        </tr>
        <tr>
            <td>
                can be multiple layers deep
            </td>
            <td>
                a child of one master detail relationship cannot be the parent of another one
            </td>
        </tr>
        <tr>
            <td>
                no impact on a security and access
            </td>
            <td>
                access to a parent determines access to a children
            </td>
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

<img src="./screenshots/screenshot-1.png" alt="screen-1.png"/>

```sql
SELECT Name, Phone, Account.Id, Account.Name FROM Contact WHERE Account.Id = '0010900000iMM6GAAW'
```

<img src="./screenshots/./screenshot-2.png" alt="screen-2.png"/>

Parent-to-child queries:

```sql
SELECT Id, Name, (SELECT Name, Phone FROM Contacts) FROM Account
```

<img src="./screenshots/./screenshot-3.png" alt="screen-3.png"/>

```sql
SELECT Id, Name, (SELECT Name, Phone FROM Contacts) FROM Account WHERE Id = '0010900000iMM6DAAW'
```

<img src="./screenshots/./screenshot-4.png" alt="screen-4.png"/>

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

# # 2 Some useful Apex code snippets

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
