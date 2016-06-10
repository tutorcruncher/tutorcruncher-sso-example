# Fernet SSO Demo

Flask and python 3 based demonstration of TutorCruncher's Single-Sign-On feature.

This is just the receiving side. Eg. users can start a session in the micro application using a 
signed and encrypted url token generated by another application eg. TutorCruncher.

The encryption and signature are accomplished using the [Fernet](https://github.com/fernet) symmetric encryption 
method (Fernet is available in numerous modern languages including python, ruby, go and javascript and 
is based on industry standard encryption algorithms).

This approach has a number of advantages:
* the token serves two purposes: it confirms that the user has come directly from the configured application
 (eg. TutorCruncher) and provides some of their credentials eg. name & role.
* more or less any data required can be passed to the SSO addon and adding new fields won't break existing addons. 
The limit here is browser url length (generally 
[2000](http://stackoverflow.com/questions/417142/what-is-the-maximum-length-of-a-url-in-different-browsers) 
characters), for reference the url in the example is 220 characters long.\
* no API integration is initially required to integrate as all required information is passed in the token.
* token expiry can easily be checked without breaking any implementation as this feature is built into Fernet. 

## Token variables

(ot strictly part of a general Fernet SSO workflow, but this is still a useful play to put the information)

TutorCruncher supplies the following variables in SSO tokens:

| Key     | Description                                                                                                                         |
|--------:|-------------------------------------------------------------------------------------------------------------------------------------|
|`id`     | id of the user, unique for each user                                                                                                |
|`nm`     | first and last name of the user                                                                                                     |
|`rt`     | role type, Administrator, Contractor (eg. tutor) or ServiceRecipient (eg. student)                                                  |
|`ts`     | unix timestamp when the user clicked the link, aka "now"                                                                            | 
|`tz`     | the user's timezone name, this maybe null if no user or branch timezone is configured **†**                                         | 
|`br_id`  | branch id, the id of the branch the user clicked the link from, generally there's one branch per company but there can be more      |
|`br_nm`  | branch name                                                                                                                         |
|`apt_id` | appointment(lesson) id, unique for each appointment **(only available when the SSO link was followed from an appointment)**         |
|`apt_nm` | name or topic of the appointment **(only available when the SSO link was followed from an appointment)**                            |
|`apt_st` | unix timestamp for the start datetime of the appointment **(only available when the SSO link was followed from an appointment)**    |
|`apt_fn` | unix timestamp for the finish datetime of the appointment  **(only available when the SSO link was followed from an appointment)**  |

**†** timezones are provided as ISO timezone names eg. `America/New_York`, see the "YZ" column 
[here](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

## Token workflow

Below is a simple example of the workflow (this is in python but the equivalent would be 
very similar in any other language):

```python
from cryptography.fernet import Fernet
import json

obj = {'rt': 'Contractor', 'nm': 'Samuel Colvin', 'id': 123, 'apt': 321}
shared_secret_key = '38mnLQKJf5J0J0Il4-wHr-KnMnjAz8cZLTvDHEAwU-c='
f = Fernet(shared_secret_key)
token = f.encrypt(json.dumps(obj))
url = 'http://www.example.com/tc-sso?token=%s' % token

# url: http://www.example.comsso-lander/testing?token=gAAAAABXBOTJZGO_-1ORdEHFSktCrUXVNNgMEIjc6IrlDyjjPzPAkn36S2-4-fKG1eFT1DlGUjAgTD3SLsO1XCgh-6MIj0x0bTYuXtrRKvu1Y6XPY8QDXWAm5B9Qr8NoThnhUZ3P36vBisUvusQaz8xQSqy26dU5rrMZ3X9YR6hdiuV-VUqM1Qw=

# on other server

f = Fernet(shared_secret_key)  # from your storage

obj2 = json.loads(f.decrypt(url.split('token=')[-1])  # should really use a proper url parser
# obj2: {u'apt': 321, u'id': 123, u'nm': u'Samuel Colvin', u'rt': u'Contractor'}
```

## Usage

This is just a simply proof of concept app which lets users "log in" and view who else has logged in.

The only extra complexity it adds which real life applications may not have to implement is a screen to set a 
new SSO shared secret to make testing easier. In general applications could set the shared secret key as a constant.

This app is running at https://fernetsso.herokuapp.com on a free dyno. **Warning:** Don't allow anything secret into
this demo server, it's not designed to be secure.

#### 1 Setup a Secret Key

You can create a new group to login to by going to https://fernetsso.herokuapp.com/add-group. You need to enter a 
valid Fernet can either key, (eg. from `Fernet.generate_key()`) if you're using TutorCruncher you can get the key from
the SSO Config list.

#### 2 Create the SSO token

You now need to created the encrypted token to login with. 

This is generally done by the SSO app TutorCruncher, however for testing you can also use the `util.py` tool 
here to generate the token, eg:

```shell
./utils.py create_token <the sso secret> "nm=Samuel Colvin" rt=Contractor id=123
> data: {"rt": "Contractor", "nm": "Samuel Colvin", "id": "123"}
> Created SSO token:
>   ?token=<the token>
```

The `utils.py create_token` script can take any arbitrary key:value pairs but this app requires the token to include
`nm` (name) and `rt` (role type).

**Note:** fernet-sso-demo as a ttl for tokens of 60 seconds so you need to be fairly nifty with your token to use it
before it expires. see [here](https://cryptography.io/en/latest/fernet/#cryptography.fernet.Fernet.decrypt) for details.

#### 3 Go to the url

You can then use the token to sign in. Go to 

```
https://fernetsso.herokuapp.com/sso-lander/<group name>?token=<the token>
```

The group name is the name you supplied in setup 1 above.

You'll be presented with a log of previous events for that group, you can reload the page to see newer events or
go to `/profile` to get a full break down of the data passed in the token about your role.
