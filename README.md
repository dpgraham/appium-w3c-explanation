# Appium W3C Explanation

* Appium 1.8.0 will be the first version of Appium to _fully_ implement [W3C](https://github.com/jlipps/simple-wd-spec)
* 1.7.2 accepts W3C parameters during session creation but coerces them into a MJSONWP session
* The [W3C](https://github.com/jlipps/simple-wd-spec) simplified spec doc that JLipps wrote explains most of how the spec works so I'd recommend reading over that first
* This document will elaborate further on how Appium specifically implements this spec

## Sessions

### New Session Request Format
  
```javascript
{
  capabilities: {
    alwaysMatch: {...},
    firstMatch: [{...}, ...}]
  }
}

// Good
// If 'firstMatch' is undefined it will be treated as a singleton array with an empty object [{}] (as per spec)
{
  capabilities: {
    alwaysMatch: {...}
  }
}

// Good
// 'mjsonwp' and 'w3c' can _both_ be provided but a W3C Session will be initiated and Appium will respond with a W3C formatted response
{
  desiredCapabilities: {...}
  capabilities: {
    alwaysMatch: {...},
    firstMatch: [{...}, ...}]
  }
}

// Good
// If 'firstMatch' is an empty array it will be treated as a singleton array with an empty object [{}] (not technically part of spec but may as well be forgiving)
{
  capabilities: {
    alwaysMatch: {...},
    firstMatch: [],
  }
}

// Good
// If 'alwaysMatch' is undefined it will be treated as an empty object {} (as per spec)
{
  capabilities: {
    firstMatch: [{...}, ...],
  }
}
```

### Capabilities Validation
* All [required parameters](https://github.com/appium/appium-base-driver/blob/973a6d70bba853de26d95c2f1f7ad8b0e1dbaffb/lib/basedriver/desired-caps.js) (platformName, deviceName) must be provided in the capabilities

```javascript
// good
// required caps are provided in 'alwaysMatch'
{
  capabilities: {
    alwaysMatch: {deviceName: 'iPhone 6s', platformName: 'iOS'},
    firstMatch: [{}]
  }
}

// good
// required caps are provided in 'firstMatch'
{
  capabilities: {
    alwaysMatch: {},
    firstMatch: [{deviceName: 'iPhone 6s', platformName: 'iOS'}]
  }
}

// good
// required caps are provided in 'alwaysMatch' and 'firstMatch'
{
  capabilities: {
    alwaysMatch: {deviceName: 'iPhone 6s'},
    firstMatch: [{platformName: 'iOS'}]
  }
}

// good
// required caps are provided in at least one of the 'firstMatch' caps
{
  capabilities: {
    alwaysMatch: {},
    firstMatch: [
      {someOtherCap: 'someOtherCap', foo: 'bar'},
      {platformName: 'iOS', deviceName: 'iPhone 6s'},
    ]
  }
}

// good
// some of the required caps are provided in alwaysMatch and the rest are in at least one of the 'firstMatch' caps
{
  capabilities: {
    alwaysMatch: {platformName: 'iOS'},
    firstMatch: [
      {someOtherCap: 'someOtherCap', foo: 'bar'},
      {deviceName: 'iPhone 6s'},
    ]
  }
}

// bad
// one of the required caps is missing
{
  capabilities: {
    alwaysMatch: {deviceName: 'iPhone 6s'},
    firstMatch: [{}]
  }
}

// bad
// the required caps are both in different firstMatch objects and not in alwaysMatch
{
  capabilities: {
    alwaysMatch: {},
    firstMatch: [
      {deviceName: 'iPhone 6s'},
      {platformName: 'iOS'},
    ],
  }
}

// bad 
// one of the required caps is invalid ('BadAutomationName' is an invalid automation name)
{
  capabilities: {
    alwaysMatch: {platformName: 'iOS', deviceName: 'iPhone 6s', automationName: 'BadAutomationName'},
    firstMatch: [{}],
  }
}
```

* The same capability can't be in `alwaysMatch` and `firstMatch`

```
// bad
// the same capability appears in firstMatch _and_ alwaysMatch
{
  capabilities: {
    alwaysMatch: {platformName: 'iOS', deviceName: 'iPhone 6s', automationName: 'BadAutomationName'},
    firstMatch: [
      {platformName: 'Android'}
    ],
  }
}

// bad
// the same capability appears in firstMatch _and_ alwaysMatch, even if they have the same value
{
  capabilities: {
    alwaysMatch: {platformName: 'iOS', deviceName: 'iPhone 6s', automationName: 'BadAutomationName'},
    firstMatch: [
      {platformName: 'iOS'} // Doesn't matter that platformName is the same in both alwaysMatch and firstMatch
    ],
  }
}
```

* The capabilities must pass [validation rules](https://github.com/appium/appium-base-driver/blob/973a6d70bba853de26d95c2f1f7ad8b0e1dbaffb/lib/basedriver/desired-caps.js)

```
// bad
// 'BadAutomationName' is not a valid automation name
{
  capabilities: {
    alwaysMatch: {platformName: 'iOS', deviceName: 'iPhone 6s', automationName: 'BadAutomationName'},
    firstMatch: [
      {platformName: 'Android'},
      {}, // Doesn't matter that this line DOES pass. It instantly fails upon finding a matching capability
    ],
  }
}

// good
// 'XCUITest' is a valid automation name
{
  capabilities: {
    alwaysMatch: {platformName: 'iOS', deviceName: 'iPhone 6s', automationName: 'XCUITest'},
    firstMatch: [{}],
  }
}

```

### Session Initiation Response

* When a session is created successfully Appium responds with this (as per spec):

```javascript
{
  value: {
    sessionId: <SESSION_ID>,
    capabilities: {...}, // Matched capabilities
  }
}
```

* Session creation example

```
// REQUEST
Body: {
  capabilities: {
    alwaysMatch: {platformName: 'iOS'},
    firstMatch: [
      {someOtherCap: "someOtherCap", foo: "bar"},
      {deviceName: "iPhone 6s"},
    ]
  }
}

// RESPONSE
HTTP Status Code: 200
Body: {
  value: {
    capabilities: {
      platformName: "iOS",
      deviceName: "iPhone 6s"
    }
  }
}
```

* If 'capabilities' are malformed or are invalid, Appium will respond with 400 Bad Request error with a message explaining the reason

  
### Prefixes

* Review official spec for indepth details (https://www.w3.org/TR/webdriver/#dfn-extension-command-prefix)

* These capabilities are 'Standard Capabilities' and ought to never be prefixed

```javascript
[
  'browserName',
  'browserVersion',
  'platformName',
  'acceptInsecureCerts',
  'pageLoadStrategy',
  'proxy',
  'setWindowRect',
  'timeouts',
  'unhandledPromptBehavior'
];
```

* Appium will accept _any_ capabilities that have the `appium:` prefix and return capabilities with the `appium:` prefix stripped out

```
// REQUEST
Body: 
{
  capabilities: {
    alwaysMatch: {platformName: 'iOS', 'appium:deviceName': 'iPhone 6s', 'appium:automationName: 'XCUITest'},
    firstMatch: [{}]
  }
}

// RESPONSE
HTTP Status Code: 200
Body: 
{
  value: {
    capabilities: {
      platformName: "iOS",
      deviceName: "iPhone 6s",
      automationName: "XCUITest",
    }
  }
}
```

* Appium will _not_ accept a request if it contains a standard capability prefixed with `appium:`. For example, this request will respond with a Bad Request error because `platformName` is a standard capability and should not be prefixed

```
// BAD REQUEST
Body: 
{
  capabilities: {
    alwaysMatch: {'appium:platformName': 'iOS', 'appium:deviceName': 'iPhone 6s', 'appium:automationName: 'XCUITest'},
    firstMatch: [{}]
  }
}
```

### W3C vs. MJSONWP

* When JSON body is passed to the session creation endpoint (`POST /session`), Appium will parse the JSON and determine if it should initiate a W3C or MJSONWP Session

* How it determines which session to run:
  * If the body contains an object called `capabilities` it will be a W3C session
  * Else if the body contains an object called `desiredCapabilities` it will be an MJSONWP session
  * If both are provided, it will be a W3C session (NOTE: If anyone objects to this and feels that it should be MJSONWP, please let me know)
