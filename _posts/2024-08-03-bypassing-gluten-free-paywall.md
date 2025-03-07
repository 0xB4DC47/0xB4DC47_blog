---
title: Bypassing the Gluten-Free Paywall
categories: [dev]
comments: true
---

A particular country takes gluten allergies very serious and thus has strict regulations for gluten-free food. Restraurants can get certified by an organization, which keeps a list of approved locations. However, access to this list requires a paid membership—either a short-term option (1 week of access) for visitors or a long-term one that requires a local tax ID. 

While planning a trip to said country, we were unable to purchase the short-term membership for unknown reasons. This lasted for about 2 weeks and we had too much planning to do so we did some exploring.   

# Analyzing Web traffic
By using burp to capture web traffic when the app opened you can see that it was making a request for the `app-update-v3.json` resource found at the following url:
`https://mobile.redacted.com/app-update-v3.json`

Analyzing the response it's clear that it's retrieving links to the updated database files:

Sample Response:
```json
{
    "version": "2",
    "databases": {
        "prontuario": {
            "version": "2024.617",
            "date": "2024-08-03T04:06:59+02:00",
            "url": "https:\/\/redacted.b-cdn.net\/redacted-apiv3\/db\/pr******io.2024.617_enc.db"
        },
        "afc": {
            "version": "2024.598",
            "date": "2024-08-03T02:11:33+00:00",
            "url": "https:\/\/redacted.b-cdn.net\/redacted-apiv3\/db\/a******fc.2024.598_enc.db"
        }
    }
}
```
Luckily for us, we can download these files without any authentication so by navigating to these URLs in a browser we can download them locally. 

We can assume these files are encrypted as we see "enc" in the name. So the next step is to find the decryption key. This would either be stored locally in the app (unsafe) or come as a response to an authenticated request to the server (good). It was the former for us..


# Finding the encryption
These encrypted database files are hosted on a CDN and seem to be updated often. By decompiling the app using `jadx-gui` we can search for keywords to find the piece of code that downloads these files. After a quick search you can find the encryption key in the `getReadableDatabase()` method of the `SQLiteDatabaseHelper` class seen below:

```java
 public synchronized SQLiteDatabase getReadableDatabase() {
     return getReadableDatabase(A******fcAfcCat****ia.dk1 + A******fcRating.dk2 + Brand.dk3 + Category.dk4 
     + InfoAg******a.dk5 + Product.dk6 + Prov******a.dk7 + Company.dk8);
 }
```

Clearly this is an attempt at obfuscating the key. However by searching for definitions you can the values for each piece of the key.


```java
A******fcAfcCat****ia.dk1 = "hMoz#kTU"
A******fcfcRating.dk2 = "BcQTbMXQ"
Brand.dk3 = "6zLxKgR4"
Category.dk4 = "kEd9kAKF"
InfoAg******a.dk5 = "fszYa8Y]g"
Product.dk6 = "uZ9QCp"
Prov******a.dk7 = "J,T4fLLBH"
Company.dk8 = "CW]BmgvH"
```

If we concat all these value we get the following key:

```java
String fullKey = "hMoz#kTUBcQTbMXQ6zLxKgR4kEd9kAKFfszYa8Y]guZ9QCpJ,T4fLLBHCW]BmgvH"
```

Having they key alone isn't enough to be able to decrypt the databases. The other important piece(s) is the encryption parameters/settings. By doing a little more reversing you'll find the constructor of the `SLiteDatabaseHelper` class. Here you can see the `cipher_compatibility` setting being set to 2:

```java
public SQLiteDatabaseHelper(String str, int i) {
     super(AicMobileApp.getInstance().getApplicationContext(), str, null, i, new SQLiteDatabaseHook() {
         /* class org.casarini.android.aicmobile.model.SQLiteDatabaseHelper.C09951 */

         @Override // net.sqlcipher.database.SQLiteDatabaseHook
         public void preKey(SQLiteDatabase sQLiteDatabase) {}

         @Override // net.sqlcipher.database.SQLiteDatabaseHook
         public void postKey(SQLiteDatabase sQLiteDatabase) {
             sQLiteDatabase.rawQuery("PRAGMA cipher_compatibility = 2;", (String[]) null).close();
         }
     });
     SQLiteDatabase.loadLibs(AicMobileApp.getInstance().getApplicationContext());
 }

```

This means that the following settings can be applied within [DB Browser for SQLite](https://sqlitebrowser.org/) to decrypt the database:

```json
"page_size": 1024
"kdf_iterations": 4000
"hmac": "SHA1"
"KDF_algorithm": "SHA1"
"plaintext_header_size: 0
```

Now we can access the full list of certified restaurants (this picture is so heavily obfuscated it could be a completely random DB):

![Image](/assets/img/gluten_free_db_decrypt.png)


## Alternative Route - Escalating Privileges by modifying shared prefs

By modifying the app's sharedpreferences, users can gain access to the list without paying. UserAccess level is checked when launching the fragment that contains the map view. This value is loaded at app load time and only ever modified from when you successfully purchase a membership. A user with root access can modify the shared preferences to escalate privileges since access controls aren't checked server-side.

Source:
```java
public void load() {
    SharedPreferences sharedPreferences;
    Context context2 = this.context;
    if (context2 != null) {
        sharedPreferences = PreferenceManager.getDefaultSharedPreferences(context2);
    } else {
        sharedPreferences = A******icMobileApp.getInstance().getDefaultSharedPreferences();
    }
    sharedPreferences.edit();
    this.name = sharedPreferences.getString("", null);
    this.surname = sharedPreferences.getString("", null);
    this.email = sharedPreferences.getString("", null);
    this.address = sharedPreferences.getString("", null);
    this.city = sharedPreferences.getString("", null);
    this.province = sharedPreferences.getString("", null);
    this.postal_code = sharedPreferences.getString("", null);
    this.telephone = sharedPreferences.getString("", null);
    this.accessLevel = sharedPreferences.getString("", null); // Vulnerable!!
    this.co*******io = sharedPreferences.getString("", null);
    Long valueOf = Long.valueOf(sharedPreferences.getLong("user.access_level_expiry", ;));
    ...
    ...
}
```
