# Authorization guidelines for easy integration with third-party apps

## Background 

The Vit@Admin app is designed to integrate well with other medical applications. The purpose the integration is to server as a bridge between other apps and NHIF (National Health Insurance Fund). In order to do that in a secure and scalable manner, other apps must comply to a specific API interface, which would allow them to integrate seemlessly and without any coding needed. Below is described the flow and API endpoints fetching various data objects. 

## Format 

All API endpoints should return a JSON object in the following format: 
```
{
	data: {},
	error: {
		status: "", //string - the HTTP status code applicable to this problem
		code: "", //string - an application-specific error code, expressed as a string value
		title: "", //string - a short, human-readable summary of the problem
		detail: "", //string - a human-readable explanation of the problem
	}
}
```

Either data or error object must be present at all times

## Authorization

For secure authentication and authorization between Vit@Admin and you app, the prefered workflow is based on OAuth 2.0's Authorization Code Grant with [Proof Key for Code Exchange](https://tools.ietf.org/html/rfc7636).

The PKCE was developer to address the threat of having the authorization code intercepted. In order to do that the client creates a code verifier (random number) and a code challenge (usually a SHA256 of the code verifier) and uses them to prove its identity.

Detailed info on the flow can be found in the link above. To summarize, the steps of the flow are the following:
1. The client (Vit@Admin app in our case) redirects the user to your domain (auth server) and sends along a code challenge
2. User enters credentials (that would usually be username and password)
3. Server validates credentials and throws error in case of failure or in case of success redirects the user back to the client (Vit@Admin app) along with an authorization code
4. Vit@Admin app exchanges a code verifier and the authorization code for an access token and a refresh token


### 1. 
Client creates a code verifier and challenge and redirects the user to the auth server.

Code Verifier -  a cryptographically random string using the characters `A-Z`, `a-z`, `0-9`, and the punctuation characters `-._~` (hyphen, period, underscore, and tilde), between 43 and 128 characters long.

Code Challenge - this is the result of using the code_challenge_method on the code_challenge, in case of SHA256, that would be the SHA256 hash of the code_verifier.
```
https://YOUR_DOMAIN/authorize?
    response_type=code //authorization code is expected
    client_id= //a unique ID for the Vit@Admin app or other third party clients, registered with your auth server
    redirect_uri= //the URL to which the user should be returned after successfull authorization
    code_challenge=<CODE_CHALLENGE> //the code challenge string
    code_challenge_method=S256 //can be either S256 or plain, but plain should not be used in production
```


### 2. 
User enters credentials (usually username and password).

### 3.
In step 3 the auth server should verify the user credentials and in case of success, it should
- generate an authorization code 
- store the code_challenge, code_challenge_method and the authorization code
- redirect the user to the redirect_uri

If the authorization request is missing the code challenge, the server should return an error response with error=invalid_request and the error_description or error_uri explaining the error.

### 4.
The client should exchange the authorization code for an access token and a refresh token. To prove that the it's still the same client that made the original call and noone intercepted the authorization code, the client sends the code_verifier and the auth server checks it against the code_challenge stored on the previous request (in case of SHA256, it hashes it and checks the hash against the challenge).
```
POST https://YOUR_DOMAIN/oauth/token
	grant_type=authorization_code //the grant type of this token request
	client_id= //a unique ID for the Vit@Admin app or other third party clients, registered with your auth server
	redirect_uri= //same as in previous request
	code=authorization_code //the authorization code received from the previous request
	code_verifier=code_verifier //the code verifier created initially by the client
```

On success the server should return a response in the following format:

```
{
	data: {
  		"access_token": "...",
  		"refresh_token": "...",
  		"id_token": "...", //optional OpenID Connect token encoding  the user's authentication information
	}
}
```

### 5. 

Refreshing the access token - if the access token has expired, the client can renew it using the refresh token provided previously by the auth server.
```
POST https://YOUR_DOMAIN/oauth/token
	grant_type=refresh_token //the grant type of this token request
	client_id= //a unique ID for the Vit@Admin app of other third party clients, registered with your auth server
	refresh_token= //the refresh token previously generated by the auth server
```

On success the server should return a response in the following format:

```
{
	data: {
  		"access_token": "...",
  		"refresh_token": "...",
  		"id_token": "...", //optional OpenID Connect token encoding  the user's authentication information
	}
}
```

## Fetching Data

In order for the Vit@Admin app to fetch all needed information on the patients file, the endpoints descried below are needed.
Each API call will require the client to authorize via [bearer token](https://tools.ietf.org/html/rfc6750), by adding the following header:
```
Authorization: Bearer <bearer_token> // the access code obtained through the API
```

### 1. Lab Results

Fetch lab results info for the currently authorized user

```
GET <YOUR_API_URI>/lab-results
```

Expected response:
```
{
	data: [{
		result_id: "" //unique identifier for the lab result entity
		date: "" // DateTime of the lab result
		assayData: [
			{
				assay: "", //string - the name of the assay
				assay_abbreviation: "", //string
				value: "", //string - the value of the assay
				unit: "", //string - unit of measurement used
				ref_range: "", //string - reference range for the value
				patho_value: "", //string - pathological value - for example - Low/High

			}, {...}
		]
	}, 
	{...},
	...]
}
```

### 2. Vital Signs

Fetch vital signs data for the currently authorized user. This data may be input by the user and can consist of many things like blood preasure, pulse, etc..

```
GET <YOUR_API_URI>/vital-signs-data
```

Expected response:
```
{
	data: [{
		id: "" //unique identifier for the vital sign measurement instance
		date: "" // DateTime of the measurement
	    vital_sign_name: "", //string - the name of what measurement is taken
	    value: "", //string - the value of the measurement
	    unit: "", //string - unit of measurement, for example "count" or "volume"
	    time_span: "", //string - optional, how long did it last?
	    frequency_cicle: "", //string - optional, frequency of measurement, for example "24h" or "week"
	    notes: "", //string - optional, free text
		
	}, 
	{...},
	...]
}
```

### 3. Doctors Appointments

Fetch all future Doctors Appointments for the currently authorized user.

```
GET <YOUR_API_URI>/appointments
```

Expected response:
```
{
	data: [{
		id: "" //unique identifier for the appointment
		date: "" // DateTime of the appointment
	    title: "", //string - free text
	    note: "", //string - free text
	    UIN: "", //string - optional, unique identification number - the physician's UIN
	    location: "", //string - location of the appointment, free text
	}, 
	{...},
	...]
}
```

Create future Doctors Appointments for the currently authorized user.

```
POST <YOUR_API_URI>/appointments
```

Body:
```
{
	data: [{
		id: "" //unique identifier for the appointment
		date: "" // DateTime of the appointment
	    title: "", //string - free text
	    note: "", //string - free text
	    UIN: "", //string - optional, unique identification number - the physician's UIN
	    location: "", //string - location of the appointment, free text
	}, 
	{...},
	...]
}
```

Expected Result:
```
{
	data: [{
		id: "" //unique identifier for the appointment
		date: "" // DateTime of the appointment
	    title: "", //string - free text
	    note: "", //string - free text
	    UIN: "", //string - optional, unique identification number - the physician's UIN
	    location: "", //string - location of the appointment, free text
	}, 
	{...},
	...]
}
```

### 4. Reminders

Fetch all future reminders that the user has set up in the app.

```
GET <YOUR_API_URI>/reminders
```

Expected response:
```
{
	data: [{
		id: "" //unique identifier for the appointment
		date: "" // DateTime of the appointment
	    title: "", //string - free text
	    note: "", //string - free text
	    location: "", //string - location of the appointment, free text
	}, 
	{...},
	...]
}
```

### 5. Treatments

Fetch all treatments that are currently active for the user, as well as all past ones.

```
GET <YOUR_API_URI>/treatments
```

Expected response:
```
{
	data: [{
		id: "" //unique identifier for the treatment
		ATCCode: "" //string - international code of the drug
		INN: "" //string - international name of the drug
	    packageSize: 0, //int - how many doses are there in a a package
	    intake_scheme: {
	    	value: 0, //int - how many units are in a single dose
	    	unit: "", //string - what's the unit of measurement, for example count, mg, ml, etc..
	    	cycle: "week", //enumeration - what's the cycle of treatment, possible values are [day, week, month]
	    	cycles_count: 0, //int - the duration of the treatment, measured in cycles
	    	star_date: "", //DateTime - the first day of the treatment (first dose to be taken)
	    	intakes: [ //an array of strings, describing the next point in time when a dose is due, depends on the cycle value
		    	{
		    		time: "05:00" //in case the cycle is set to "day", the intake can only be time in 24:00h format
		    	},
		    	{
		    		time: "07:30", //in case the cycle is set to "week", time is optional
		    		day: "Wednesday" //in case the cycle is set to "week", the day field must contain a valid day of the week
		    	}, {
		    		date: "24.05.2020", //in case the cycle is set to "month", the date field must contain a valid date string
		    		time: "20:00" //in case the cycle is set to "month", time is optional
		    	}
	    	]
	    }
	}, 
	{...},
	...]
}
```

### 6. Diagnoses

Fetch all diagnoses for the current user

```
GET <YOUR_API_URI>/diagnoses
```

Expected response:
```
{
	data: {
		date: "", //DateTime
		id: "", //String - unique ID of the diagnosis in the system
		url: "", //String - a valid URL that allows for a PDF document of the diagnosis to be downloaded indefinitely.
	}
}
```
