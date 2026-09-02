---

title: 'Using Python to Harness the Power of the Fortinet APIs'
description: 'How I used the FortiSASE REST API and Python to turn endpoint and VPN session data into useful, real-time operational insight.'
pubDate: 'Sep 02 2026'
----------------------

A management portal is great when a human wants to look at something once.

An API is considerably better when a machine needs to look at it every five minutes.

I had a good example of this recently during a FortiSASE deployment.

We needed to be able to answer, on demand, two fairly simple questions:

* How many customer endpoints enrolled into FortiClient EMS were currently reporting online?
* Of those online endpoints, how many currently had an active FortiSASE tunnel protecting their traffic?

None of this information was particularly difficult to obtain.

The FortiSASE portal already contained it.

The problem was how we wanted to use it.

Logging into a management portal every few minutes, navigating between views, correlating figures manually and dealing with authentication and session timeouts wasn't particularly appealing.

What I wanted instead was something I could ask at any point:

**What does our FortiSASE endpoint coverage look like right now?**

And get an answer.

That is where the API came in.

## Fortinet's APIs

Fortinet exposes REST APIs across a wide range of its products and cloud services.

API specifications are available through the FortiAPI section of the Fortinet Developer Network (FNDN), which provides resources for developing applications and automation around Fortinet products.

That might mean a custom portal, an automated deployment or provisioning system, a reporting tool or simply a script that removes a repetitive administrative task.

One thing worth knowing before heading there is that FNDN access requires registration and sponsorship by Fortinet employees.

For this particular project, I was interested in the **FortiSASE REST API**.

FortiSASE provides APIs that allow administrators and applications to retrieve operational information as well as interact with supported configuration resources.

In my case, I didn't want to change anything.

I just wanted data.

## The Use Case

The important distinction here is that I wasn't looking for a single API call that returned:

> Your FortiSASE coverage is 63.87%.

That would have been too easy.

Instead, the information I wanted existed across different parts of the platform.

I needed information about:

1. The endpoints known to EMS and their current status.
2. The active FortiSASE VPN sessions.

I could then correlate those two datasets myself.

And once you start thinking about an API in those terms, things become considerably more interesting.

The portal isn't necessarily the finished product anymore.

It becomes a **data source**.

Python can retrieve that data, transform it and present an entirely new view tailored to the operational question you actually want to answer.

## Why Python?

A lot of API documentation and examples naturally lend themselves to tools such as Postman.

I like Postman.

It provides a friendly interface, makes requests and responses easy to visualise and is a great way of learning how an unfamiliar REST API behaves before you start writing code around it.

For this use case, though, I wanted something different.

I wanted to:

* Make several API requests automatically.
* Handle pagination where required.
* Correlate the returned datasets.
* Calculate my own statistics.
* Identify individual endpoints requiring attention.
* Produce structured JSON that something else could consume.
* Eventually run the whole thing without anyone touching it.

Python was an obvious fit.

Using Python with Fortinet's REST APIs isn't anything new. Fortinet themselves publish examples showing Python interacting with products such as FortiGate.

The interesting part is what you choose to build with it.

## Step 1: Create an API User

Before calling the FortiSASE API, I needed an API identity.

At a high level, this involved:

1. Logging into FortiCloud Identity & Access Management (IAM) with an appropriately privileged account.
2. Creating the required permission profile.
3. Creating an API user and assigning those permissions.
4. Downloading the credentials generated for that API user.

Those credentials can then be used to request an OAuth access token.

This is an important separation.

My automation isn't logging in using my own administrator account, nor is it trying to automate an interactive MFA login.

It has its own identity and, importantly, its own permissions.

## Step 2: Authenticate

Using the API user's credentials, Python can request an OAuth access token from Fortinet.

My authentication function looked broadly like this:

```python
def authenticate(api_id, password):
    auth_body = {
        "username": api_id,
        "password": password,
        "client_id": "FortiSASE",
        "client_secret": "",
        "grant_type": "password",
    }

    response = requests.post(
        AUTH_URL,
        json=auth_body,
        timeout=30,
    )

    response.raise_for_status()
    data = response.json()

    return data["access_token"]
```

The response contains a Bearer access token which can then be supplied with subsequent API requests.

It also contains information such as the token lifetime and a refresh token, allowing an application to account for token expiry rather than assuming authentication lasts forever.

Already, this has an obvious advantage over trying to automate a normal administrator session.

The process has been designed for machine-to-machine access.

## Step 3: Get the Data

For my particular use case, two FortiSASE Monitor API resources were of interest:

```python
BASE_URL = "https://portal.prod.fortisase.com"

ENDPOINTS_URL = (
    f"{BASE_URL}/monitor-api/v1/endpoints/details"
)

VPN_SESSIONS_URL = (
    f"{BASE_URL}/monitor-api/v1/user/vpn/sessions"
)
```

Rather than dumping everything into one enormous function, I broke the process into smaller pieces.

For example:

```python
get_all_endpoints(session)
get_vpn_sessions(session)
get_endpoint_username(endpoint)
build_status(endpoints, vpn_sessions)
```

The first functions dealt with retrieving information from FortiSASE.

The more interesting work happened afterwards.

## Turning API Data into Information

At this point I effectively had two datasets.

One represented the endpoints known to EMS.

Another represented active FortiSASE VPN sessions.

Python could now correlate them.

From that, I could build a summary along the lines of:

```json
{
    "totalEndpoints": 250,
    "managedEndpoints": 245,
    "unmanagedEndpoints": 5,
    "emsOnlineEndpoints": 155,
    "emsOfflineEndpoints": 90,
    "activeVpnSessions": 99,
    "protectedEndpoints": 99,
    "noAgentSessionEndpoints": 56,
    "coveragePercent": 63.87
}
```

*The values shown here are illustrative rather than customer data.*

Those numbers are far more useful operationally than two unrelated screens in a portal.

In this example, I can immediately see that 155 endpoints are currently reporting online to EMS, but only 99 have an active FortiSASE session.

More importantly, I don't just know that there is a difference of 56.

Because I still have the underlying endpoint data, I can produce the actual list:

```json
"affectedEndpoints": [
    {
        "host": "LAPTOP-001",
        "user": "user@example.com"
    },
    {
        "host": "LAPTOP-002",
        "user": "another.user@example.com"
    }
]
```

That changes the conversation significantly.

Instead of:

**"Our coverage looks to be around 64%."**

I can say:

**"These are the endpoints currently reporting online to EMS without an associated active FortiSASE session."**

That is something an engineer can investigate.

## Make the Output Reusable

I didn't want the Python console to become another management portal that somebody had to stare at.

So the script produces structured JSON.

Alongside the endpoint information, I also include when the dataset was generated:

```python
"generatedAtUtc": (
    datetime.now(timezone.utc)
    .isoformat(timespec="seconds")
    .replace("+00:00", "Z")
)
```

That's a small detail, but an important one.

If you're looking at operational information, you need to know **when it was true**.

Once the result exists as JSON, the Python script has done its job.

Anything capable of consuming JSON can now make use of it.

And this is where I decided to take the idea a little further.

## From Python Script to Operational Dashboard

I had a Windows Server available running IIS.

So rather than leaving the output as a JSON file, I used it as the data source for a small web dashboard.

The overall flow became:

**FortiSASE → REST API → Python → JSON → Web Dashboard**

The dashboard could then show things such as:

* Total managed and unmanaged endpoints.
* EMS online versus offline endpoints.
* Active FortiSASE sessions.
* Percentage coverage.
* Endpoints online without an active session.
* The time at which the data was last refreshed.

The Python process could be run on demand or triggered on a schedule, meaning the web page became an automatically refreshed operational view rather than a manually assembled report.

And that's probably a subject for another article.

## This Is Where APIs Become Powerful

None of what I've described involved doing something the FortiSASE portal couldn't fundamentally do.

That's almost the point.

APIs don't only become valuable when you need to perform some huge automated configuration change.

Sometimes their value comes from taking data that already exists and using it differently.

The same pattern can be applied far beyond this example.

Retrieve information from multiple sources.

Correlate it.

Apply your own logic.

Store it.

Graph it.

Alert on it.

Feed it into another system.

Run it every five minutes instead of asking an engineer to check it every five minutes.

Once you start treating infrastructure platforms as sources of structured data rather than purely graphical interfaces, there are a lot of interesting problems you can solve.

## A Few Things I Wouldn't Skip

API access is powerful, and that makes some basic controls particularly important.

### Use least privilege

Give an API user only the permissions required for the task.

A monitoring script shouldn't have unrestricted administrative access simply because that makes the permissions easier to configure.

### Protect the credentials

Don't hardcode API usernames, passwords or tokens into scripts, particularly scripts that might later end up in source control.

Use an appropriate secrets store, operating-system credential mechanism or another protected method of supplying them at runtime.

### Expect tokens to expire

Authentication should be treated as part of the application lifecycle.

Handle token expiry and refresh appropriately rather than assuming that a token obtained today will work indefinitely.

### Set sensible timeouts and handle errors

Networks fail.

APIs return errors.

Tokens expire.

Services occasionally respond slowly.

A production script should expect those things rather than crashing mysteriously when they happen.

### Handle pagination

A successful response containing the first page of results isn't necessarily the complete dataset.

This becomes particularly important as endpoint counts grow.

### Be careful with write operations

My use case was read-only.

If you're using an API to make configuration changes, apply considerably more caution: use least privilege, validate what you're changing, keep an audit trail and use meaningful change descriptions or comments where the target resource supports them.

### Test before production

If the operation can change infrastructure, prove the logic against a lab, test tenant or controlled subset of devices first.

And, perhaps most importantly:

**Never disable TLS certificate validation just because it makes the Python error disappear.**

## Where Next?

For me, this started with a very straightforward requirement:

*Tell me how many endpoints are online and how many of them are actually protected by FortiSASE.*

The FortiSASE portal had the raw information.

The REST API made that information accessible programmatically.

Python allowed me to correlate it and turn it into something more useful.

And a little bit of web development turned the result into an operational dashboard that could sit there updating itself throughout the day.

That's the bit I enjoy about automation.

It doesn't necessarily have to replace an engineer or orchestrate thousands of configuration changes.

Sometimes it just removes a repetitive job and gives the engineer better information.

I'll go into the dashboard side of this in more detail in a future post.

In the meantime, if you're doing something interesting with the Fortinet APIs — or you've got a FortiSASE/API problem that doesn't quite fit what the portal gives you out of the box — feel free to get in touch.

There is usually more data sitting behind that GUI than you might think.
