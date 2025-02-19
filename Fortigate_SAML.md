#Fortigate SAML Setup

## Planning

1. Figure out your WAN interface, this will be critical for several commands that will need to be ran. 


## Variables

To make documentation easier, I am using several varriables which can be referenced in thi table. 

| Variable | What is it for?|
|-----------|---------------|
| $VPN_Tunnel | This is the name of the IPSec VPN tunnel that you made. |
| $WAN_Interface | This is the interface that connects to the interent, and is the termination point for IPSec.|
| $SAML_IKE_PORT | This is the port that the Fortigate will respond on to do the authentication.(If you have no preference, use 9443)|
| $Fortigate_FQDN | The Fully qualified domain name of your Fortigate's external interface.  |
| $SAML_Config_Fortigate | This is the name of the configuration on your Fortigate to connect to Entra. ex. Entra-SAML-VPN-Auth |
| $IPSec_VPN_Name | This is the name of the configuration on the Fortigate of your IPSec Tunnel. |



Below is a list of terms used in FortiGate GUI, and their equivalents in Azure, and the required SAML attributes:

| FortiGate GUI | Azure                   |
|----------|----------------------------|
| IdP entity ID   | Entra ID Identifier |
| IdP single sign-on URL      | Login URL                 |
| IdP single logout URL    | Logout URL                      |
|   SP entity ID       | Identifier (Entity ID)       |
| SP ACS (login) URL     |       Reply URL (Assertion Consumer Service URL)                     |
| SP SLS (logout) URL      | Logout URL               |
|SP portal URL|Sign on URL


## Step By Step for SAML Login

1. Create a new Enterprise application in Entra ID. Go to MicrosoftEntra ID -> Enterprise applications -> Create New Application -> FortiGate SSL VPN -> Name -> Create.
2. In the newly created application, select Set up a single sign-on, and select SAML.
3. Open the Fortigate, go to User & Authentication > Single Sign-On and create a new connection. 
4. Copy the Entity ID, Assertion Consumer URL, and Single Logout Service URL and enter them into the Basic SAML Configuration Screen on Entra.
5. Close out of the Basic SAML Configuration Screen
6. Copy the Login URL, Entra Identifier, and Logout URL back into the screen on the Fortigate. 
7. Export the Certificate in Base64 from Entra.
8. Upload the certificate to the fortigate.
9. Start with sections #3 and #4. In section #3, download the certificate. In section #4 copy all three values (they will be used in step 5). Keep this page open, it is needed at a later step to finish the configuration.
10. Switch to the FortiGate. The first step is to import the Entra ID SAML certificate from the previous step.
    - In GUI: System -> Certificates -> Import -> Remote Certificate.
11. FortiGate SAML configuration. Go to Security Fabric -> Fabric Connectors -> Security Fabric Setup -> Single Sign-On Settings.


## [Step by Step for Forticlient Configuration](https://docs.fortinet.com/document/fortigate/7.6.1/administration-guide/432396/configuring-microsoft-entra-id-as-saml-idp-and-fortigate-as-saml-sp#:~:text=This%20topic%20discusses%20the%20configurations%20steps%20required%20if,for%20FortiClient%20remote%20access%20dialup%20IPsec%20VPN%20clients.)

### Microsoft Entra

1. Create the Application in Entra
2. Choose from the Entra Gallery, "Fortinet SSL VPN" We will rename this when we create it. 
3. Open the newly created Application, click on "Users and Groups"
4. Add the User or Group that you want to test with. If you want to allow all users to access the VPN, under "Properties" change the "Assignment Required" to "No"
5. Configure single sign-on:
   1. Browse to Enterprise application > All applications > FortiGate IPsec VPN application.
   2. In the Manage section, select Single sign-on.
   3. In the Select a single sign-on method page, select SAML.
   4. In the Set up Single Sign-On with SAML page, select Edit for Basic SAML Configuration to edit the settings:
      1. In Identifier, enter a URL in the pattern https://<FortiGate IP or FQDN address>:<Custom SAML-IKE port>/remote/saml/metadata
      2. In Reply URL, enter a URL in the pattern https://<FortiGate IP or FQDN address>:<Custom SAML-IKE port>/remote/saml/login
      3. In Sign on URL, enter a URL in the pattern https://<FortiGate IP or FQDN address>:< Custom SAML-IKE port >/remote/saml/login
      4. In Logout URL, enter a URL in the pattern https://<FortiGate IP or FQDN address>:< Custom SAML-IKE port >/remote/saml/logout
   5. Click Save
6. The Fortigate expects some attributes with specific names, those can be found below. (With my version of FortiOS, Groups are not supported so I didn't configure those.)
   1. Go to the Attribute & Claims Section
   2. Click "Edit"
   3. Select Add new claim.
   4. For Name, enter username.
   5. For Source attribute, select user.userprincipalname.
   6. Select Save.
7. Next we need to configure the Certificate
   1. In the Certificate Section, click Edit
   2. Add an email address that will be notified when the certificate is going to expire.
   3. Click Save
   4. Select the Base64 version of your certificate, download this to your computer.
8. Scroll down to section 4 "Set Up"
   1. Copy the Login URL, Microsoft Entra Identifier, and Logout URL. We will need these for the Fortigate Setup.

### Fortigate

1. Login to your Fortigate
   1. On the FortiGate, go to User & Authentication > Single Sign-On > Create new.
   2. Enter the Name as $SAML_Config_Fortigate.
   3. In the Address field, enter the FQDN/IP information in the following format:
      
     \$Fortigate_FQDN:\$SAML_IKE_PORT

   4. In Service Provider Configuration, verify the following URLs (Entity ID, Assertion consumer service URL, Single logout service URL) are correct in your Enterprise application > All applications > FortiGate IPsec VPN > Single sign-on page > Basic SAML Configuration.
   5. Back on the Fortigate, click "Next"
   6. In Identity Provider Details, set the Type as Custom
   7. Paste the URLs copied from last step of section To configure Enterprise application on Azure portal: Follow the mapping at the top of this document.
   8. On Certificate, Create a new one, and upload the Base64 encoded certificate that we downloaded earlier.
   9. In the Additional SAML section add "username" to the "Attribute used to identify users" field.
   10. Click "Submit"
   11. Go to User & Authentication > User Groups > Create New.
   12. Enter Name as SAML-ENTRA-ID-Group.
   13. In Remote Groups, click Add.
   14. From the Remote Server dropdown, select $SAML_Config_Fortigate SAML server.
   15. In the groups section select "Any". (This is a bug in several versions of FortiOS. You can't limit it to a specific group, this is the work around. You can still control access by selecting users in the Entra Users & Groups section.)
   16. Click "OK" then "OK"
2.  Use the FortiGate CLI to bind and associate the SAML server with the VPN gateway interface (port1) as follows:

```
config system interface
    edit "$WAN_Interface"
        set ike-saml-server "$SAML_Config_Fortigate"
    next
end
```

3. Set the IKE port in the CLI,

```
config system global
    set auth-ike-saml-port $SAML_IKE_PORT
end 
```
4. Create your new VPN Tunnel Config
   1. Go to VPN > IPsec Tunnels.
   2. Click Create New > IPsec Tunnel. The VPN Creation Wizard is displayed.
   3. Enter the Name as $IPSec_VPN_Name.
   4. Configure the Template type as Custom.
   5. Click "Next"
   6. Configure the following:

| | |
|--|--|
| Name | FCT_SAML |
|Comments|(Optional)
|Network
|IP Version|IPv4
|Remote Gateway|Dialup User
|Interface|port1
|Mode Config|Enable
|Use system DNS in mode config|(Optional) Enable FortiClient to use the host's DNS server after it connects to VPN.
|Assign IP From|Enable|Select Address/Address Group from the dropdown list.
|IPv4 mode config
|Client Address Range|VPN_Client_IP_Range
|VPN_Client_IP_Range is configured from 10.212.134.1 to 10.212.134.200. If it is not already created, select Create > Address from the dropdown menu to create a new address object. See Subnet for more information.
|Subnet Mask|255.255.255.255
|DNS Server|1.1.1.1
|Authentication
|Method|Pre-shared key
|IKE
|Version|2
|Peer Options
|Accept Types|Any peer ID
|Phase 1 Proposal
|Encryption|AES128
|Authentication|SHA256

   7. Click the "OK" button to save it.
   8. You should now see the tunnel under the VPN tab.
   9. Open the CLI on the Fortigate and run the following. 

```
config vpn ipsec phase1-interface
    edit "$IPSec_VPN_Name"
        set eap enable
        set eap-identity send-request
    next
end
```
### Firewall Policy

We can't forget to set the policy!

1. Go to Policy & object
2. Create a new Policy
   1. Name - VPN Network Access
   2. Incoming Interface - $IPSec_VPN_Name
   3. Outgoing Interface - $LAN
   4. Source - Select your VPN Range, Under Users, select your Group
   5. Destination - Select the Internal Hosts that you want to access over VPN.
   6. Service - Any, or limit as you find necessary.
