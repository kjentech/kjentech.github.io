# ADC Gateway Configuration

This ADC config will give you:
- ADC Gateway
- LDAP + RADIUS MFA
	- in this example, both SMS and push notifications are supported
	- If configured (see [ADC Customizations](ADC Customization.md)), push notification users will get a message asking them to check their app
- RDP Proxy
	- with and without SSO, example can be seen under Traffic Policies

Internal IP 10.10.10.10 is NATed to external IP x.x.x.x
If these configuration steps are done from top to bottom, you have the complete setup.
replace "domain.com" with your domain.

Visual customizations: See [ADC Customizations](ADC Customization.md).

### RDP Client profiles
RDP Client Profile: rd_client_noextras

RDP Client Profile: rd_client_noextras_multimon
- Multiple Monitor Support: ENABLE

RDP Client Profile: rd_client_copyclipboard
- Redirect Clipboard: ENABLE
- Redirect Drives: ENABLE
- Redirect Printers: ENABLE
- Audio Capture Mode: ENABLE
- Video Playback Mode: ENABLE

Common:
- RDP Host: gateway.domain.com

### RDP Server profiles
RDP Server Profile: rdp_serverprof_3389_1
- Port: 3389
- RDP Redirection: ENABLE

Session Profile: vpn_rd_noextras
- RDP Client Profile: rd_client_noextras

Session Profile: vpn_rd_noextras_multimon
- RDP Client Profile: rd_client_noextras_multimon

Session Profile: vpn_rd_copyclipboard
- RDP Client Profile: rd_client_copyclipboard

Common:
- Clientless Access: On
- Advanced Clientless VPN Mode: DISABLED
- Single Sign-on to Web Applications: On
- Single Sign-on with Windows: Off
- Client Choices: Off
- Default Authorization Action: ALLOW
- ICA Proxy: OFF
- Single SIgn-on Domain: domain.com


### Session policies
Session Policy: vpn_rd_noextras_pol
- Expression: true
- Profile: vpn_rd_noextras

Session Policy: vpn_rd_noextras-multimon_pol
- Expression: AAA.USER.IS_MEMBER_OF("netscaler-Multimon")
- Profile: vpn_rd_noextras_multimon

Session Policy: vpn_rd_copyclipboard_pol
- Expression: AAA.USER.IS_MEMBER_OF("netscaler-ClipboardAndDrives")
- Profile: vpn_rd_copyclipboard

Session Policy: vpn_rd_copyclipboard_pol2
- Expression: true
- Profile: vpn_rd_copyclipboard


### Traffic profiles
Traffic Profile: trafficAct_SSO
- Protocol: HTTP
- Single SIgn-on: ON

Traffic Profile: trafficAct_noSSO
- Protocol: HTTP
- Single Sign-on: OFF

Traffic Policy: trafficPol_SSO (will be applied to all objects with low priority)
- Request Profile: t_act1
- Expression: http.req.url.contains("/rdpproxy")

Traffic Policy: trafficPol_noSSO
- Request Profile: trafficact_noSSO
- Expression: http.req.url.contains("/rdpproxy")

Traffic Policy: trafficPol_noSSO_exception (will only be applied to this one server with high priority)
- Request Profile: trafficact_noSSO
- Expression: http.req.url.contains("/rdpproxy/exceptionRd.domain.com")


### LDAP server
LDAP Server: ldap_dc01

- Server Name: dc01.domain.com
- Security Type: TLS
- Port: 389
- Base DN: dc=domain,dc=com
- Administrator Bind DN: netscalerAD@domain.com
- Administrator password: `Pa$$w0rd`
- Server Logon Name Attribute: userPrincipalName
- Search Filter: blank
- Group Attribute: memberOf
- Sub Attribute Name: cn
- SSO Name Attribute: SamAccountName
- Email: mail
- Nested Group Extraction: Enabled
- Maximum Nesting Level: 5
- Group Search Filter: blank
- Group Name Identifier. cn
- Group Search Attribute: memberOf
- Group Search Sub-Attribute: cn


### RADIUS server
RADIUS Server: radius_mfa01

- Server Name: mfa01.domain.com
- Port: 1812
- Secret Key: see NPS
- Time-Out: 60
- NAS ID: MFA
- Password Encoding: pap


### Authentication Policies
Authentication Policy: authpol_ldap_dc01
- Expression: true
- Request Server: ldap_dc01

Authentication Policy: authpol_radius_mfa01
- Expression: true
- Request Server: radius_mfa01

Login Schema Profile: loginSchema_custom
- Authentication schema: /nsconfig/loginschema/SingleAuth_custom.xml
    -   SingleAuth_custom.xml:
```
<?xml version="1.0" encoding="UTF-8"?><AuthenticateResponse xmlns="http://citrix.com/authentication/response/1">
            <Status>success</Status>
            <Result>more-info</Result>
            <StateContext/>
            <AuthenticationRequirements>
            <PostBack>/nf/auth/doAuthentication.do</PostBack>
            <CancelPostBack>/nf/auth/doLogoff.do</CancelPostBack>
            <CancelButtonText>Cancel</CancelButtonText>
            <Requirements>
            <Requirement><Credential><ID>login</ID><SaveID>ExplicitForms-Username</SaveID><Type>username</Type></Credential><Label><Text>Email address:</Text><Type>nsg-login-label</Type></Label><Input><AssistiveText>...@domain.com</AssistiveText><Text><Secret>false</Secret><ReadOnly>false</ReadOnly><InitialValue/><Constraint>.+</Constraint></Text></Input></Requirement>
            <Requirement><Credential><ID>passwd</ID><SaveID>ExplicitForms-Password</SaveID><Type>password</Type></Credential><Label><Text>Password:</Text><Type>nsg-login-label</Type></Label><Input><Text><Secret>true</Secret><ReadOnly>false</ReadOnly><InitialValue/><Constraint>.+</Constraint></Text></Input></Requirement>
            <Requirement><Credential><ID>loginBtn</ID><Type>none</Type></Credential><Label><Type>none</Type></Label><Input><Button>Log On</Button></Input></Requirement>
            </Requirements>
            </AuthenticationRequirements>
            </AuthenticateResponse>
```

- Enable Single Sign On Credentials: On


### nFactor flows
nFactor Flow: nFactorFlow_LDAP-MFA

![nfactor](img-136-8a6f5550043340d098da8980f487ac72.png)
- nFactorFlow_ldap
    -   Login Schema: loginSchema_custom
    -   Policy: authpol_ldap_dc01 (100, Next)
- azuremfa
    -   Login Schema: blank
    -   Policy: authpol_radius_mfa01 (100, End)


### AAA vServers
AAA vServer: authserv_radius-mfa
- IP: 0.0.0.0:0 (Non-Addressable)
- Portal Thene: CUSTOM-RfWebUI
- nFactor Flow: nFactorFlow_LDAP-MFA


### Authentication Profile
Authentication Profile: AuthProf_gateway_LDAP-MFA
- Authentication Virtual Server: authserv_radius-mfa
- Authentication Host: gateway.domain.com
- Authentication Domain domain.com
- Authentication Level: 0

### Gateway vServers
Gateway vServer: vserver_gateway

- IP: 0.0.0.0:0 (Non-Addressable)
- RDP Server Profile: rdp_serverprof_3389_1
- DTLS: false
- Session Policy: vpn_rd_noextras-multimon_pol (Priority 100)
- Session Policy: vpn_rd_noextras_pol (Priority 50000)
- Authentication Profile: AuthProf_gateway_LDAP-MFA
- Portal Theme: CUSTOM-RfWebUI

### Content Switching
Content Switching Action: csact_gateway
- Target Gateway vServer: vserver_gateway

Content Switching Policy: cspol_gateway
- Expression: HTTP.REQ.HOSTNAME.CONTAINS("gateway")
- Action: csact_gateway

Content Switching vServer: cs_gateway
- IP: 10.10.10.10:443
- Content Switching Policy: cspol_gateway

Responder Action: http_to_https_actn
- Expression: "https://" + HTTP.REQ.HOSTNAME.HTTP_URL_SAFE + HTTP.REQ.URL.PATH_AND_QUERY.HTTP_URL_SAFE
- Response Status Code: 302

Responder Policy: http_to_https_pol
- Expression: HTTP.REQ.IS_VALID
- Action: http_to_https_actn

Content Switching vServer: http-https_redirector
- IP: 10.10.10.10:80
- Responder Policy: http_to_https_pol


### Bookmarks
Add bookmarks as needed.
- Address: rdp://RdServer.domain.com
- Use Gateway as reverse proxy: YES



## Customizations
Portal Theme: CUSTOM-RfWebUI

Customizations: See [ADC Customizations](ADC Customization.md).

Examples include:
- Enhanced Authentication Feedback
- strings.en.json
    -   "Bad password" -> "User name or password is incorrect"
    -   "Processing your request" -> "Please approve the notification in your Authenticator app"
- script.js
    -   customAuthTop, Bottom, Footer
- theme.css
- plugins.xml
    -   defaultView = Desktops




## Flow
1. User browses to http://gateway.domain.com with HTTP
2. Request hits Content Switching vServer "http-https_redirector" -> HTTP 302 -> https://gateway.domain.com (port 443)
3. Request hits Content Switching vServer "cs_gateway" -> Gateway vServer "vserver_gateway"
4. Gateway vServer "vserver_gateway" has Authentication Profile "AuthProf_gateway_LDAP-MFA" -> AAA vServer "authserv_radius-mfa"
5. AAA vServer "authserv_radius-mfa" has Portal Theme "CUSTOM-RfWebUI" and nFactor Flow "nFactorFlow_LDAP-MFA" with first factor "nFactorFlow_ldap", has Login Schema "loginSchema_custom.xml"
6. User is shown webside with organization logo (via Portal Theme) and Email Address/Password fields (via Login Schema)
7. Second factor in nFactor "azuremfa" has no Login Schema, user is shown spinner with message about Authenticator or is shown text field for SMS.
8. User is shown Gateway portal with Desktops chosen.

