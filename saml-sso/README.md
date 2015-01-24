# OpenAM SSO Configuration

This is a quick "get started" to testing out SAML SSO with OpenAM bosh-lite.  It is a work in progress and not necessarily optimal.

Step by step docs on setting up the Login server with OpenAM [are here](https://github.com/cloudfoundry/login-server/blob/master/docs/OpenAM-README.md) ; though that assumes an isolated integration test.  This guide modifies those instructions for a bosh-lite install.

This presumes you're running bosh-lite with the standard configuration (192.168.50.1 for your machine, 192.168.50.4 for the vagrant machine, 10.244.0.34 for the CF load balancer)

###Step 1
Download and install software

- a) Download/Unzip a Tomcat .tar.gz from [http://tomcat.apache.org](http://tomcat.apache.org) (On Mac OS X, you can use `brew install tomcat7`, which puts tomcat in `/usr/local/opt/tomcat7/libexec`)
- b) Download OpenAM WAR file  [https://backstage.forgerock.com/#!/downloads/enterprise/OpenAM](https://backstage.forgerock.com/#!/downloads/enterprise/OpenAM) (look for OpenAM 12 non subscription)
- d) Create a directory under `webapps/` called 'openam'
- e) Extract contents (zip) of OpenAM-12.0.0.war into `webapps/openam`
- f) ensure you have the appropriate Java version setup for Tomcat:  OpenAM 12.0 supports Java 8; OpenAM 11.0 requires Java 7


**Note** OpenAM 11.0.0 GA has a bug with verifying XML signatures in later (v22+) JDK 7 releases.  See https://bugster.forgerock.org/jira/browse/OPENAM-2673 and https://bugster.forgerock.org/jira/browse/OPENAM-3920 ; The 11.x point releases require a Forgerock subscription; use OpenAM 12 - or if feely lucky, use an older JDK 7 (v21).


###Step 2
Initialize OpenAM

- a) Start Tomcat (`bin/catalina.sh`).
- b) Go to [http://openam.192.168.50.1.xip.io:8080/openam](http://openam.192.168.50.1.xip.io:8080/openam)
- c) Click 'Create Default Configuration' and set password for amAdmin and UrlAccessAgent
- d) Click Create
(this will create the directory ~/openam
  if you wish to restart an installation, wipe this dir clean and restart tomcat)
  - e) Log in as amAdmin and the password you just created

###Step 3
Setup OpenAM as an Identity Provider (IDP)

- a) Click "Create Hosted Identity Provider"
- b) Select 'test' for the signing key
- c) Type 'circleoftrust' for "New Circle of Trust" (value is not used by us)
- d) Click 'Configure'  , then 'Finish'
- e) Click on the 'Federation' tab, and then click on the Entity Provider 'http://openam.192.168.50.1.xip.io:8080/openam'
- f) In the 'NameID Value Map' section, add `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified=mail`
- g) Delete the blank `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified=` entry
- h) Click 'Save'

###Step 4
Modify your bosh-lite deployment
- a) Update your bosh-lite deployment with the manifest.yml file which enables SAML assertions with the included keypair:  `bosh deployment ./manifest.yml ; bosh deploy`
- b) This should restart your login and uaa jobs

###Step 5
Import certificate into OpenAM
- a) Import cert.pem from this repo into `~/openam/openam/keystore.jks` via `keytool -import -trustcacerts -alias cfsaml -file cert.pem -keystore ~/openam/openam/keystore.jks -storepass changeit`
- b) Bounce the OpenAM Tomcat instance

###Step 6
Configure OpenAM to have login-server as a service that wishes to authenticate

- a) On the 'Common Tasks' tab, click 'register a remote service provider'
- b) Put 'http://login.10.244.0.34.xip.io/saml/metadata' as the URL
- c) Click 'Configure'


###Step 7
Create a SAML user

- a) Click 'Access Control'
- b) Click '/ (Top Level Realm)'
- c) Click 'Subjects'
- d) Click 'New'
Enter user information -
After the user is created, click on it again, and give the user an email address
- e) **Important** Log out of OpenAM


###Step 8
Test SAML Authentication

- a) Go to http://login.10.244.0.34.xip.io/login
- b) Click "Sign in with OpenAM"
- c) Sign in with the user you created


###Step 9
Test CF SSO Authentication
- a) with the Cloudfoundry CLI, try `cf login --sso`
- b) go to http://login.10.244.0.34.xip.io/passcode for your one time passcode
- c) enter the passcode in your CLI terminal

**Note:**  If if you login with a "domain username" with OpenAM, CF uses your email address as your actual username.   This likely is configurable with some exploration.

**Note 2:** The SAML process should automatically create new users in UAA, so they can login to CF, but they won't have any roles assigned.   An administrator needs to `cf set-space-role` for this user at minimum to allow this user to `cf push` applications

# Debugging
- `bosh ssh login_z1`, and `tail -f /var/vcap/sys/log/login/login.log`
- `bosh ssh uaa_z1`, and `tail -f /var/vcap/sys/log/uaa/uaa.log`
- `tail -f ~/openam/openam/debug/Federation`
- `tail -f ~/openam/openam/log/SAML2.access`
- `tail -f ~/openam/openam/log/SAML2.error`
- If tweaking `manifest.yml` for bosh-lite , some config template changes don't lead to a restart of the login job, you may want to `bosh restart login_z1 0` to kick it

# Unresolved issues

* CF login server should validate IDP key, haven't looked at this yet
* CF login-server seems to insist on a NameID of  `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified` of which the value must be an email address, no matter what nameID I put in the manifest under `saml` or `saml:providers:openam-local`, might be worth digging into further
* Web Access Management for actual buildpacks with OpenAM requires remote registration of the OpenAM agent at start time, this is in theory doable but could create a mess of agents in OpenAM if there's no garbage collection of dead agents due to failures or (de)scaling
* Test with legacy OpenSSO, OpenAM 12
