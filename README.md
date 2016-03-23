## Migrate Your Push Notifications from Parse to CleverTap

### Sign up for CleverTap

[Sign up here](https://clevertap.com/sign-up/).  It's free, no credit card required.

### Import your Parse users

[Automatically import your Parse users](https://dashboard.clevertap.com/x/import/parse.html) from your CleverTap dashboard.

### Add CleverTap to your app

* [iOS integration guide](https://support.clevertap.com/integration/ios-sdk/)  

* [Android integration guide](https://support.clevertap.com/integration/android-sdk/)  

* [Server API guide](https://support.clevertap.com/server/overview/)  


#### Maintaining Continuity of Users During the Transition

When you import your Parse users into CleverTap, a CleverTap user profile will be created for each of your Parse users.  Those profiles will include the corresponding push tokens and channel values (and other custom property values) from your Parse user.

Our SDK will be able to match the imported CleverTap user profile with a device user via the Parse Installation Id.  To enable this matching, you must add the following flag to your Info.plist/AndroidManifest.xml file when integrating the relevant CleverTap SDK.

`AndroidManifest.xml`:

```xml
<meta-data
    android:name="CLEVERTAP_PARSE"
    android:value="1"/>
```    

`Info.plist`:  Add a key called `CleverTapParse` as a `BOOLEAN truthy` value in your `.plist file`.

With a successful matching, your current user channels and other custom profile properties will be available to your client code.

#### Android GCM Token Considerations

The default with Parse was to collect Android GCM tokens using the Parse GCM Sender Id.  As a result, those tokens will no longer work (when you switch to a new provider or Parse ceases operations).  If you do not have tokens generated with your own GCM Sender Id, our suggestion is to run CleverTap and Parse side by side in your app for a reasonable time period during which you can generate new tokens using your own GCM Sender Id.   

Please refer to our [Android Push Notification guide](https://support.clevertap.com/integration/android-sdk/#handling-multiple-push-notification-providers) for more information.

#### Managing Channels

Your users' current Parse channels will be imported and sync'd to the corresponding CleverTap user profile and will be available to your client-side code.

##### Android

###### Accessing Channels

```java
clevertap = CleverTapAPI.getInstance(getApplicationContext());
JSONArray channels = (JSONArray ) clevertap.profile.getProperty("channels");
if (channels != null) {
    for (int i=0; i <channels.length(); i++) {
        try {
            Log.d("Channel", "value is " + channels.get(i));
        } catch (JSONException e) {
            // no-op
        }
    }
}
```

###### Adding or Removing Channels

```java
clevertap = CleverTapAPI.getInstance(getApplicationContext());
clevertap.profile.addMultiValueForKey("channels", "mets");
```

```java
clevertap = CleverTapAPI.getInstance(getApplicationContext());
clevertap.profile.removeMultiValueForKey("channels", "yankees");
```

##### iOS

###### Accessing Channels

```objc
NSArray *channels = [[CleverTap sharedInstance] profileGet:@"channels"];
if (channels && [channels isKindOfClass:[NSArray class]]) {
    for (NSString *channel in channels) {
        NSLog(@"channel is %@", channel);
    }
}
```

###### Adding or Removing Channels

```objc
[[CleverTap sharedInstance] profileAddMultiValue:@"yankees" forKey:@"channels"];
```

```objc
[[CleverTap sharedInstance] profileRemoveMultiValue:@"mets" forKey:@"channels"];
```

#### Sending Push Notifications

##### From the CleverTap Dashboard

You can create and manage push notification campaigns directly in your [CleverTap Dashboard->Campaigns->Push](https://dashboard.clevertap.com/x/push/notification.html).  

##### Via the CleverTap Server API.  

[Learn more about the CleverTap Server API here](https://support.clevertap.com/server/overview/).

In this example we are creating a push notification to be sent to users subscribed to the "yankees" and "mets" channels.  

Endpoint: https://api.clevertap.com/1/targets/create.json  

**Sample Request**  

    curl -H "Content-Type:application/json"
         -H "X-CleverTap-Account-ID:xxx-xxx-xxx"
         -H "X-CleverTap-Passcode:xxx-xxx-xxx"
         --data "@payload"
         "https://api.clevertap.com/1/targets/create.json"

**Payload**

```json
{
    "name": "My API Campaign",
    "where": {
        "common_profile_prop": {
            "profile_fields": [
                {
                    "name": "channels",
                    "value": ["yankees", "mets"]
                }
            ]
        }
    },
    "content":{
        "title": "Hi!",
        "body": "How are you doing today?",
        "platform_specific": {
            "ios": {
                "deep_link": "example.com",
                "sound_file": "example.caf",
                "category": "reactive",
                "badge_count": 1,
                "key": "value_ios"
            },
            "android": {
                "background_image": "http://example.jpg",
                "default_sound": true,
                "deep_link": "example.com",
                "key": "value_android"
            }
        }
    },
    "devices": [
        "android",
        "ios",
    ],
    "when": "now"
}
```

**Response**

```json
{
    "status": "success",
    "id": 1457433898,
    "estimates": {
        "android": 90,
        "ios": 300
    }
}
```
[

##### From Cloud Code

If you are running your own instance of [parse-server](https://github.com/ParsePlatform/parse-server), you can use the CleverTap Node.js module to send push notifications.  Please [see our fork of the parse-server-example server](https://github.com/CleverTap/parse-server-example) for more details and usage. 

* Install the CleverTap Node module: `npm install --save clevertap`

* Add your CleverTap Account ID and CleverTap Account Passcode as environment variables (CLEVERTAP_ACCOUNT_ID and CLEVERTAP_ACCOUNT_PASSCODE, respectively).  

You can then enable CleverTap in your Cloud Code ([see our example cloud/main.js implementation](https://github.com/CleverTap/parse-server-example/blob/master/cloud/main.js)) to send push like this, for example:

```javascript
const CleverTap = require("clevertap");
const clevertap = CleverTap.init(process.env.CLEVERTAP_ACCOUNT_ID, process.env.CLEVERTAP_ACCOUNT_PASSCODE);

Parse.Cloud.define('push', function(req, res) {
    
    /**
     * Send an immediate push to users subscribed to the specified channels
     * Callable from the Parse SDK Cloud Code methods
     */

    const channels = req.params.channels;

    if (!channels) {
        res.error("channels not present");
        return;
    }

    const payload = {
        "name": "test push " + Math.round(new Date().getTime()/1000),
        "when": "now",
        "where": {
            "common_profile_prop": {
                "profile_fields": [{"name": "channels", "value": channels}]
            }
        },
        "content": {
            "title":"Hello!",
            "body":"Just testing"
        },
        "devices": ["ios", "android"]
    }

    clevertap.targets(clevertap.TARGET_CREATE, payload, {"debug":1}).then((r) => {
        res.success(r);
    });
});

Parse.Cloud.afterSave('GameScore', function(req) {
    /**
     * Send all users an immediate push notifiying them of a new Game Score. 
     *
     */

    const gameScore = req.object;
    const body = (gameScore) ? `New Game Score: ${gameScore.get("playerName")} ${gameScore.get("score")}` : "New Game Score";
    const payload = {
        "name": "test push " + Math.round(new Date().getTime()/1000),
        "when": "now",
        "segment":"all",
        "content": {
            "title": "Hey There!",
            "body": body,
        },
        "devices": ["ios", "android"]
    }

    clevertap.targets(clevertap.TARGET_CREATE, payload, {"debug":1}).then((r) => {
        console.log(r);
    });
});
```
