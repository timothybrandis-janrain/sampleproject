+++
date = "2016-10-25T15:01:57-07:00"
title = "API-Based Implementation"

[menu]
  [menu.top_nav_menu]
    name = "API-Based Implementation Guide"
    parent = "registration"
    weight = 30

+++

# API-Based Implementation

The purpose of this section is to guide developers through the authentication process using the Janrain Registration Server and supporting server-side API (Application Programming Interface) endpoints without using the Janrain Registration Widget or Mobile Libraries.

All code examples in this document will be demonstrated as cURL calls or using generic PHP code. In most cases, cURL calls should be submitted to the API endpoints over SSL using the HTTP POST Method. Additionally, the /oauth example calls will not accept query parameters as everything must be submitted as a url-encoded form parameter.

# Traditional Login and Registration

## Traditional Login

Complete traditional login via the [oauth/auth_native_traditional](/api/registration/authentication/#oauth-auth_native_traditional) call with required fields from the sign-in form.

```php
$api_call = '/oauth/auth_native_traditional';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',
    'redirect_uri' => 'https://localhost',

    // the name of your sign-in form as defined in the flow file
    'form' => 'signInForm',

    // required fields from signInForm
    'signInEmailAddress' => $_POST['email'],
    'currentPassword' => $_POST['password']
);

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | New access token is returned |
| User not found / Incorrect username or password (`invalid_credentials`) | Provide paths for [Traditional Registration]({{< relref "#traditional-registration">}}) and [Forgot Password]({{< relref "#forgot-password">}}) |

## Traditional Registration

Complete traditional registration via the [oauth/register_native_traditional](/api/registration/authentication/#oauth-register_native_traditional) call with required fields from the registration form.

```php
$api_call = '/oauth/register_native_traditional';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSIONS,
    'locale' => 'en-US',
    'redirect_uri' => 'https://localhost',

    // the name of your registration form as defined in the flow file
    'form' => 'registrationForm',

    // required fields from registrationForm
    'firstName' => $_POST['firstName'],
    'lastName' => $_POST['lastName'],
    'displayName' => $_POST['displayName'],
    'emailAddress' => $_POST['email'],
    'newPassword' => $_POST['password'],
    'newPasswordConfirm' => $_POST['passwordConfirm'],
);

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | User record is created and new access token is returned |
| Email address is already being used | Prompt user to authenticate again using the login method they previously used, or attempt to use the [Forgot Password]({{< relref "#forgot-password">}}) feature |

# Social Login and Registration

## IDP Authentication

The first step to complete a social login or registration is to authenticate with the IDP. This can be done in one of two ways:

  - [Via the social login widget]({{< relref "#idp-authentication-(widget)">}})
  - [Via link to the social login application (no widget)]({{< relref "#idp-authentication-(no-widget)">}})

### IDP Authentication (Widget)

Add the social login widget to your login page. See [Implementing Social Login](/social/implementing-sl/#implementing-social-login) for full implementation instructions, including how to get the code to add the widget to your page.

Once the social login widget is properly implemented, the user can simply click one of the rendered buttons in order to authenticate with the IDP.

### IDP Authentication (No Widget)

For more flexibility, you can create your own social login buttons that link to your social login application. There should be a different link for each social provider. The following is an example using Google+.

```php
<a href="https://my-app.rpxnow.com/googleplus/start?language_preference=en&token_url=https://my-token-url&display=popup&applicationId=abcdefghijklmnopqrst">Sign in with Google+</a>
```

The following contains the detials for a few Social providers

| Providers | URL |
| --------- | ----------- |
| googleplus | `https://{application domain}/googleplus/start?token_url={token url}` |
| facebook | `https://{application domain}/facebook/connect_start?token_url={token url}&ext_perms=publish_stream,email,offline_access` |


## Social Login

Once you have the social login token, the next step is to attempt to authenticate the user via the [oauth/auth_native](/api/registration/authentication/#oauth-auth_native) call. You'll pass the social login token into the call in the `token` parameter.

PHP example:

```php
$api_call = '/oauth/auth_native';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',
    'redirect_uri' => 'https://localhost',
    'registration_form' => 'socialRegistrationForm',

    // social login token obtained from previous step
    'token' => $_POST['token']
);
$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | New access token is returned |
| User not found (`310`, `record_not_found`) | Continue with [Social Registration]({{< relref "#social-registration">}}) |
| User already exists with that email address (`380`, `email_address_in_use`) | Continue with [Account Merge]({{< relref "#account-merge">}}) |
| Invalid social login token (`invalid_argument`) | Provide a resolution path for this error |

## Social Registration

If the previous oauth/auth_native call returns a `310` error (`record_not_found`), initiate social registration using the [oauth/register_native](/api/registration/authentication/#oauth-register_native) endpoint. You'll pass the social login token into the call in the `token` parameter.

PHP example:

```php
$api_call = '/oauth/register_native';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',
    'response_type' => 'token',
    'redirect_uri' => 'https://localhost',
    'form' => 'socialRegistrationForm',

    // required fields from social registration form
    'firstName' => $_POST['firstName'],
    'lastName' => $_POST['lastName'],
    'displayName' => $_POST['displayName'],
    'emailAddress' => $_POST['email'],

    // social login token obtained from previous steps
    'token' => $_POST['token']
);
$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
$accessToken = $api_response->{'access_token'};
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | User record is created and new access token is returned |
| Email address is already being used | Provide a resolution path for this error<br /> _**Note:** This error can occur when the user has an existing record and attempts to login with a different social provider that does NOT return a verified email address_ |
| Invalid social login token (`invalid_argument`) | Provide a resolution path for this error |

## Thin Social Registration

Thin registration is a configuration option that determines the behavior of the [oauth/auth_native](/api/registration/authentication/#oauth-auth_native) call when a new user authenticates. If the parameter is set to `true`, a new record will be created immediately. If set to `false` or omitted from the call, you will need to complete social registration using the [oauth/register_native]({{< relref "#social-registration">}}) call demonstrated above.

PHP example:

```php
$api_call = '/oauth/auth_native';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',
    'redirect_uri' => 'https://localhost',
    'registration_form' => 'socialRegistrationForm',

    // enable thin social registration
    'thin_registration' => 'true',

    // social login token obtained from previous step
    'token' => $_POST['token']
);
$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | User record is created and new access token is returned |
| User already exists with that email address (`380`, `email_address_in_use`) | Continue with [Account Merge]({{< relref "#account-merge">}}) |
| Invalid social login token (`invalid_argument`) | Provide a resolution path for this error |

# Account Merge

When an [oauth/auth_native](/api/registration/authentication/#oauth-auth_native) call fails with a `380` error (`email_address_in_use`), the next step is to initiate the merge process. You can prompt the user to confirm that they'd like to merge this social account with their existing user record. 

When merging accounts, there are two scenarios to consider:

  - Merge social account with existing social record
  - Merge social account with existing traditional record

Each of these two scenarios is handled differently.

## Merge Account with Existing Social Record

1. If the `existing_provider` value returned in the `380` error response is a social provider (e.g. `"facebook"`), the user must authenticate with that provider to prove ownership of the existing account. (See [IDP Authentication]({{< relref "#idp-authentication">}}) above.) You'll use the returned social login token in the next step.

2. To merge accounts, make an [oauth/auth_native](/api/registration/authentication/#oauth-auth_native) call that passes in a token and a "merge" token. 

  - The social login token for the _existing_ social provider is passed into the `token` parameter. 
  - The social login token for the _new_ social provider is passed into the `merge_token` parameter.

```php
$api_call = '/oauth/auth_native';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',
    'redirect_uri' => 'https://localhost',

    // social login token for existing social account 
    'token' => $_POST['token']

    // social login token for new social account
    // (must be the same token from the previous failed oauth/auth_native call)
    'merge_token' => $_GET['merge_token'],
);

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | Account is merged and new access token is returned |

## Merge Account with Existing Traditional Record

If the `existing_provider` value returned in the `380` error response is `"capture"`, make an [oauth/auth_native_traditional](/api/registration/authentication/#oauth-auth_native_traditional) call that passes in a "merge" token. 

  - The user must provide the login credentials for their existing traditional account.
  - The social login token for the new social provider will be passed into the `merge_token` parameter.

```php
$api_call = '/oauth/auth_native_traditional';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',
    'redirect_uri' => 'https://localhost',

    // the name of your sign-in form as defined in the flow file
    'form' => 'signInForm',

    // required fields from signInForm
    'signInEmailAddress' => $_POST['email'],
    'currentPassword' => $_POST['password'],

    // social login token for new social account
    // (must be the same token from the previous failed oauth/auth_native call)
    'merge_token' => $_POST['merge_token']
);

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | Account is merged and new access token is returned |

# Account Link/Unlink

Account linking/unlinking requires a valid Janrain access token (from a previous Authentication or Registration). It is not possible to link one existing user record to another, so the only scenario that needs to be addressed is linking to a Social account. 

For both linking and unlinking, it will be necessary to make an additional API call to the [entity](/api/registration/entity/#entity) endpoint to verify the current list of the user's linked accounts.

## Account Link

1. Authenticate with the social provider to retrieve the social login token. (See [IDP Authentication]({{< relref "#idp-authentication">}}) above.)

2. Link the social account via the [oauth/link_account_native](/api/registration/authentication/#oauth-link_account_native) call.

```php
$api_call = '/oauth/link_account_native';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',
    'redirect_uri' => 'https://localhost',

    // valid access token
    'access_token' => $_SESSION['access_token'],

    // social login token obtained from previous step
    'token' => $_POST['token']
);

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | Account is linked |
| Social account already exists (`unique_violation`) | Provide a resolution path for this error |

## Account Unlink

Unlink a social account via the [oauth/unlink_account_native](/api/registration/authentication/#oauth-unlink_account_native) call.

```php
$api_call = '/oauth/unlink_account_native';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',

    // valid access token
    'access_token' => $_SESSION['access_token'],

    // identifier value from profiles.[profile].identifier schema attribute
    'identifier_to_remove' => $_POST['identifier']
);

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | Account is unlinked |

# Email Verification

The [oauth/verify_email_native](/api/registration/authentication/#oauth-verify_email_native) endpoint will trigger the Registration system to send an email based on the configuration defined for the form used in the API call.

Email verification can also be initiated when a user registers (via oauth/register_native_traditional).

**1. Send verification email**

```php
$api_call = '/oauth/verify_email_native';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',

    // page where the user is sent
    'redirect_uri' => EMAIL_VERIFICATION_URL,

    // the name of your resend-verification form as defined in the flow file
    'form' => 'resendVerificationForm',

    // required field from resendVerificationForm
    'signInEmailAddress' => $_POST['email']
);

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | Email verification email is sent to user |
| No account found for provided email address (`invalid_credentials`) | Provide a resolution path for this error |
| Email address already verified (`triggered_error`) | Provide a resolution path for this error |

**2. Retrieve the verification code**

A successful oauth/verify_email_native call will send the email to the user which contains the `verify_email_url` appended with a verification code. The landing page for this link must parse the verification code and consume it via the [access/useVerificationCode](/api/registration/authentication/#access-useverificationcode) API call.

**3. Consume the verification code**

```php
curl -X POST

// verification code parsed from verify_email_url
--data-urlencode `verification_code=12345678912345678912345678912345` \

'https://my-app.janraincapture.com/access/useVerificationCode'
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | Janrain Registration server automatically sets the TimeStamp on the user's `emailVerified` attribute |
| Verification code not recognized (`invalid_argument`) | Provide a resolution path for this error |

# Forgot Password

The [oauth/forgot_password_native](/api/registration/authentication/#oauth-forgot_password_native) endpoint will trigger the Registration system to send an email based on the configuration defined for the form used in the API call. 

A unique constraint of this API call is that the code that is generated must be used with a widget or [oauth/token](/api/registration/authentication/#oauth-token) API call that is configured with the same API Client ID that was used to initiate the API call.

**1. Send reset password email**

```php
$api_call = '/oauth/forgot_password_native';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',

    // page where the user is sent
    'redirect_uri' => PASSWORD_RECOVER_URL,

    // the name of your forgot-password form as defined in the flow file
    'form' => 'forgotPasswordForm',

    // required field from forgotPasswordForm
    'signInEmailAddress' => $_POST['email']
);

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | Password recover email is sent to user |
| No account found with that email address (`no_such_account`) | Provide a resolution path for this error |
| Account is social only _(If applicable; depends on your flow configuration)_ | Provide a resolution path for this error |

**2. Retrieve the authorization code**

Parse the authorization code from the `password_recover_url`.

**3. Exchange the authorization code for an access token**

Via the [oauth/token](/api/registration/authentication/#oauth-token) call. **This should be done server-side.**

```php
$api_call = '/oauth/token';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,

    // client secret which matches the client_id above
    'client_secret' => JANRAIN_LOGIN_CLIENT_SECRET,

    // page where the user is sent
    'redirect_uri' => PASSWORD_RECOVER_URL,

    'grant_type' => 'authorization_code',

    // authorization code parsed from password_recover_url
    'code' => $_GET['code']
);

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);

// Store the access token in a variable so that it can be added to the
// change password form as a hidden form element.

if ($api_response->stat == "ok") {
    $access_token = $api_response->access_token;
}
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | Access token is returned, continue to next step |

**4. Reset the password**

Use the [oauth/update_profile_native](/api/registration/authentication/#oauth-update_profile_native) call to submit a new password using the `changePasswordFormNoAuth` form (Note: this is the default form name in the standard configuration).

```php
$api_call = '/oauth/update_profile_native';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'access_token' => $_POST['access_token'],
    'locale' => 'en-US',
    'form' => 'changePasswordFormNoAuth',

    // required fields from changePasswordFormNoAuth form
    'newPassword' => $_POST['new_password'],
    'newPasswordConfirm' => $_POST['confirm_password']
);

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

# Profile Update

## Update Profile Data

An [entity](/api/registration/entity/#entity) call can be made to retrieve the user information to pre-populate the Edit Profile fields. The user information is then updated via the [oauth/update_profile_native](/api/registration/authentication/#oauth-update_profile_native) endpoint.

```php
$api_call = '/oauth/update_profile_native';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',
    'access_token' => $accessToken,

    // required form = editProfileForm
    'form' => 'editProfileForm',

    // profile field(s) to update
    'firstName' => $_POST['firstName'],
    'lastName' => $_POST['lastName'],
    'displayName' => $_POST['displayName'],
    'emailAddress' => $_POST['email']
);
$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | User's record is updated |
| Birthdate is not a valid date (`invalid_form_fields`) | The birthdate field is a special field that must be sent as three (3) different field parameters: `birthdate[dateselect_year]`, `birthdate[dateselect_month]`, `birthdate[dateselect_day]` |
| Other field validation error (`invalid_form_fields`) | The pertinent error message(s) defined in the Registration configuration will be returned. Provide validation message(s) to user so that they may correct these values and try again.  |

## Change Password

A logged-in user can change their password directly via the [oauth/update_profile_native](/api/registration/authentication/#oauth-update_profile_native) endpoint and the changePasswordForm. Note that this form and workflow is different than the "forgot password" implementation.

```php
$api_call = '/oauth/update_profile_native';
$params = array(
    'client_id' => JANRAIN_LOGIN_CLIENT_ID,
    'flow' => JANRAIN_FLOW_NAME,
    'flow_version' => JANRAIN_FLOW_VERSION,
    'locale' => 'en-US',
    'access_token' => $accessToken,

    // required form = changePasswordForm
    'form' => 'changePasswordForm',

    // profile field(s) to update
    'currentPassword' => $_POST['current_password'],
    'newPassword' => $_POST['new_password'],
    'newPasswordConfirm' => $_POST['confirm_password'],
);
$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, JANRAIN_CAPTURE_URL.$api_call);
curl_setopt($curl, CURLOPT_POST, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
curl_setopt($curl, CURLOPT_POSTFIELDS, http_build_query($params));
$api_response = json_decode(curl_exec($curl));
curl_close($curl);
```

| Response | Outcome / Next Step |
| --------- | ----------- |
| Success (`ok`) | User's record is updated with new password |






