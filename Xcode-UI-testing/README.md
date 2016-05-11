### INTRODUCTION

[Colo for iOS in the App Store](https://appsto.re/us/mxIWR.i)

Colo has an Xcode UI testing suite implemented in Swift. I recently wanted to find out how much of Colo's first-time user scenarios I could automate. When someone launches Colo for the first time, they are asked if the app may access their contacts and their location. These two authorization requests have two possible outcomes, and two possible "permissions disabled" messages may be displayed if either of the authorization requests were declined. Those messages may appear on the three main screens within the app. These factors create 24 distinct verification tests to perform.

To test these scenarios correctly, the simulator or device must have the contacts and location permissions unset for my app. Can this default state of privacy settings be enforced with automation? As of Xcode 7.3.1, I do not see a way to accomplish this. I requested it via an Apple bug report, although I can understand that Apple may be hesitant to add such a feature. The default privacy settings scenario can be achieved manually, either via the Settings > General > Reset > Reset Location & Privacy menu item in the Settings app, or via the Simulator > Reset Content and Settings... menu item in the iOS Simulator.

The need for manual intervention is unfortunate, but I've decided that there is value in automating the rest of the tests. Every user experiences these code paths, so they need to be flawless.

### IMPLEMENTATION

Xcode 7's UI automation has a handy way of dealing with alerts: the `addUIInterruptionMonitorWithDescription` function. Since the first-time user scenario involves two distinct alerts, with slightly different button labels, my tests distinguish between them with the alert's `label`. I've implemented this as two separate UI interruption monitors. I'll use the contacts alert handler as the example:

    addUIInterruptionMonitorWithDescription("contacts authorization") { (alert) -> Bool in
        if ("“Colo” Would Like to Access Your Contacts" == alert.label) {
            alert.buttons["OK"].tap()
            return true
        }
        return false
    }

The above example always chooses the "OK" button for the contacts authorization alert. If the UI interruption doesn't match the expected `alert.label`, no action is taken and false is returned. The next step was to create a parameter for the button text, so either option could be selected as required for a particular test. So the UI interruption monitor was moved into its own function:

    func respondToContactsAlert(buttonText: String) {
        // UI interruption monitor code goes here
    }

And the alert button tap changes from

    alert.buttons["OK"].tap()

to

    alert.buttons[buttonText].tap()

When I want to accept contacts authorization, the `buttonText` will be "OK", and when I want to decline, `buttonText` will be "Don’t Allow". Here's the updated function:

    func respondToContactsAlert(buttonText: String) {
        addUIInterruptionMonitorWithDescription("contacts authorization") { (alert) -> Bool in
            if ("“Colo” Would Like to Access Your Contacts" == alert.label) {
                alert.buttons[buttonText].tap()
                NSLog("handled contacts alert")
                return true
            }
            return false
        }
    }

With my two handler functions implemented (the location alert handler is the same except for the string values), I added them to a test method. But the test didn't work and the first alert was not handled. Why? Because the first alert appeared before the interruption monitor was added. The solution was to move the interruption handler functions from the test function to `setUp()`. But that meant that the `setUp()` implementation was tied to only one test case, that is, only one combination of choosing yes or no for both the contacts and location alerts.

The next step was to divide the test class into four subclasses, one for each set of responses to the two permissions alerts. Each subclass would have only one test function, responsible for visiting each of the three screens in the app and verifying whether Colo's "permission disabled" messages were visible as appropriate.

I also wanted to ensure that both alerts were shown and handled, otherwise the tests are invalid. So I added two `Bool` variables to my base class, called `contactsAlertAppeared` and `locationAlertAppeared`, initialized to `false`, and set to `true` inside the respective UI interruption monitors:

    alert.buttons[buttonText].tap()
    NSLog("handled contacts alert")
    self.contactsAlertAppeared = true

The base class for all of these test subclasses then tests these `Bool` values in its `tearDown()` function:

    override func tearDown() {
        // Ensure that both alerts appeared
        XCTAssertTrue(contactsAlertAppeared)
        XCTAssertTrue(locationAlertAppeared)
        super.tearDown()
    }

### RESULTS

These automated tests revealed a layout constraint failure due to one of the items being nil. This is important for two reasons: it's something that can be investigated and fixed for the next update, and justifies the decision to automate these user scenarios.

### CONCLUSION

I've automated everything possible for 24 important UI test cases, and discovered a UI layout problem that needs to be fixed. Every time I prepare a new release of Colo, or make major UI changes, I can rely on these tests to verify these important first-time user scenarios.

Xcode UI testing could do more to enable this kind of testing: either allow tests to run in a sandbox, or allow tests to reset privacy settings. For now, I'll perform the manual step before running these tests. And these test classes will be excluded from my UI Tests target so all of the enabled tests are expected to pass.
