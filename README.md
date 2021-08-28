# PactSwift

[![Build](https://github.com/surpher/PactSwift/actions/workflows/build.yml/badge.svg)](https://github.com/surpher/PactSwift/actions/workflows/build.yml)
[![codecov](https://codecov.io/gh/surpher/PactSwift/branch/main/graph/badge.svg)][codecov-io]
[![MIT License](https://img.shields.io/badge/license-MIT-green.svg?style=flat)][license]
[![PRs Welcome!](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)][contributing]
[![slack](http://slack.pact.io/badge.svg)][pact-slack]
[![Twitter](https://img.shields.io/badge/twitter-@pact__up-blue.svg?style=flat)][pact-twitter]

<p align="center">
  <img src="Documentation/images/pact-swift.png" width="350" alt="PactSwift logo" />
</p>

This framework provides a Swift DSL for generating [Pact][pact-docs] contracts.

It implements [Pact Specification v3][pact-specification-v3] and runs the mock server "in-process". No need to set up any specific mock services or extra tools 🎉.

## Installation

Note: see [Upgrading][upgrading] for notes on upgrading and breaking changes.

### Swift Package Manager

#### Xcode

1. Enter `https://github.com/surpher/PactSwift` in [Choose Package Repository](./Documentation/images/08_xcode_spm_search.png) search bar
2. Use minimum version `0.8.x` when [Choosing Package Options](./Documentation/images/09_xcode_spm_options.png)
3. Add `PactSwift` to your [test](./Documentation/images/10_xcode_spm_add_package.png) target. Do not embed it in your application target.

#### Package.swift

```sh
dependencies: [
    .package(url: "https://github.com/surpher/PactSwift.git", .upToNextMinor(from: "0.8.0"))
]
```

#### Linux
<details><summary>Linux Installation Instructions</summary>

When using `PactSwift` on a Linux platform you will need to compile your own `libpact_ffi.so` library for your Linux distribution from [pact-reference/rust/pact_ffi][pact-reference-rust]. It is important you build the version of `libpact_ffi.so` that builds the same header files as provided by `PactMockServer`. See [`PactMockServer`](https://github.com/surpher/PactMockServer) [release notes](https://github.com/surpher/PactMockServer/releases) for details.

See `/Scripts/build_libpact_ffi` for some inspiration building libraries from Rust code.

When building and testing your project you can either set `LD_LIBRARY_PATH` pointing to the folder containing your `libpact_ffi.so`:

```sh
export LD_LIBRARY_PATH="/absolute/path/to/your/rust/target/release/:$LD_LIBRARY_PATH"
swift build
swift test -Xlinker -L/absolute/path/to/your/rust/target/release/
```

or you can move your `libpact_ffi.so` into `/usr/local/lib`:

```sh
mv /path/to/target/release/libpact_ffi.so /usr/local/lib/
swift build
swift test -Xlinker -L/usr/local/lib/
```
</details>

### Carthage

```sh
# Cartfile
github "surpher/PactSwift" ~> 0.8
```

```sh
carthage update --use-xcframeworks
```

**NOTE:**
- `PactSwift` is intended to be used in your [test target](./Documentation/images/11_xcode_carthage_xcframework.png).
- If running on `x86_64` (Intel machine) see [Scripts/carthage][carthage_script] ([#3019-1][carthage-issue-3019-1], [#3019-2][carthage-issue-3019-2], [#3201][carthage-issue-3201])

## Generated Pact contracts

By default, generated Pact contracts are written to `/tmp/pacts`. Define a `PACT_OUTPUT_DIR` environment variable (in [`Run`](./Documentation/images/12_xcode_scheme_env_setup.png) section) with the path to directory you want your Pact contracts to be written into.

Sandboxed apps (macOS) are limited in where they can write Pact contract files into. The default location is the `Documents` folder in the sandbox (eg: `~/Library/Containers/xyz.example.your-project-name/Data/Documents`). Setting the environment variable `PACT_OUTPUT_DIR` might not work without some extra settings tweaks. Look at the logs in debug area for the Pact file location.

## Writing Pact tests

- Instantiate a `MockService` object by defining [_pacticipants_][pacticipant],
- Define the state of the provider for an interaction (one Pact test),
- Define the expected `request` for the interaction,
- Define the expected `response` for the interaction,
- Run the test by making the API request using your API client and assert what you need asserted,
- Share the generated Pact contract file with your provider (eg: upload to a [Pact Broker][pact-broker]),
- Run [`can-i-deploy`][can-i-deploy] (eg: on your CI/CD) to deploy with confidence.

### Example Test

```swift
import XCTest
import PactSwift

@testable import ExampleProject

class PassingTestsExample: XCTestCase {

  static var mockService: MockService!

  override class func setUp() {
    super.setUp()
    mockService = MockService(consumer: "Example-iOS-app", provider: "some-api-service")
  }

  // MARK: - Tests

  func testGetUsers() {
    // #1 - Define the API contract by configuring how `mockService`, and consequently the "real" API, will behave for this specific API request we are testing here
    PassingTestsExample.mockService

      // #2 - Define the interaction description and provider state for this specific API request that we are testing
      .uponReceiving("A request for a list of users")
      .given(ProviderState(description: "users exist", params: ["first_name": "John", "last_name": "Tester"])

      // #3 - Define the request we promise our API consumer will make
      .withRequest(
        method: .GET,
        path: "/api/users",
        headers: nil, // `nil` means we (and the API Provider) should not care about headers.
        body: nil // same as with headers
      )

      // #4 - Define what we expect `mockService`, and consequently the "real" API,
      // to respond with for this particular API interaction we are testing
      .willRespondWith(
        status: 200,
        headers: nil, // `nil` means we don't care what the headers returned from the API are.
        body: [
          // We will use matchers here, as we normally care about the types and structure, not necessarily the actual value.
          "page": Matcher.SomethingLike(1),
          "per_page": Matcher.SomethingLike(20),
          "total": ExampleGenerator.RandomInt(min: 20, max: 500),
          "total_pages": Matcher.SomethingLike(3),
          "data": Matcher.EachLike(
            [
              "id": ExampleGenerator.RandomUUID(), // We can also use example generators
              "first_name": Matcher.SomethingLike("John"),
              "last_name": Matcher.SomethingLike("Tester"),
              "salary": Matcher.DecimalLike(125000.00)
            ]
          )
        ]
      )

    // #5 - Fire up our API client
    let apiClient = RestManager()

    // Run a Pact test and assert **our** API client makes the request exactly as we promised above
    PassingTestsExample.mockService.run(timeout: 1) { [unowned self] mockServiceURL, done in

      // #6 - _Redirect_ your API calls to the address MockService runs on - replace base URL, but path should be the same
      apiClient.baseUrl = baseURL

      // #7 - Make the API request.
      apiClient.getUsers() { users in

          // #8 - Test that **our** API client handles the response as expected. (eg: `getUsers() -> [User]`)
          XCTAssertEqual(users.count, 20)
          XCTAssertEqual(users.first?.firstName, "John")
          XCTAssertEqual(users.first?.lastName, "Tester")
        }

        // #9 - Notify MockService we're done with our test, else your Pact test will time out.
        done()
      }
    }
  }

  // More tests for other interactions and/or provider states...
  func testCreateUser() {
    PassingTestsExample.mockService
      .uponReceiving("A request to create a user")
      .given(ProviderState(description: "user does not exist", params: ["first_name": "John", "last_name": "Appleseed"])
      .withRequest(
        method: .POST,
        path: Matcher.RegexLike("/api/group/whoopeedeedoodah/users", term: #"^/\w+/group/([a-z])+/users$"#),
        body: [
          // You can use matchers and generators here too, but are an anti-pattern.
          // You should be able to have full control of your requests.
          "first_name": "John",
          "last_name": "Appleseed"
        ]
      )
      .willRespondWith(
        status: 201,
        body: [
          "identifier": Matcher.FromProviderState(parameter: "userId", value: .string("123e4567-e89b-12d3-a456-426614174000")),
          "first_name": "John",
          "last_name": "Appleseed"
        ]
      )

   let apiClient = RestManager()

    PassingTestsExample.mockService.run { mockServiceURL, done in
     // trigger your network request and assert the expectations
     done()
    }
  }
  // etc.
}
```

`MockService` holds all the interactions between your consumer and a provider. For each test method, a new instance of `XCTestCase` class is allocated and its instance setup is executed.
That means each test has it's own instance of `var mockService = MockService()`. Hence the reason we're using a `static var mockService` here to keep a reference to one instance of `MockService` for all the Pact tests. Alternatively you could wrap the `mockService` into a singleton.  
Suggestions to improve this are welcome! See [contributing][contributing].

References:

- [Issue #67](https://github.com/surpher/PactSwift/issues/67)
- [Writing Tests](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/testing_with_xcode/chapters/04-writing_tests.html#//apple_ref/doc/uid/TP40014132-CH4-SW36)

## Provider verification

<details><summary>Verification options</summary>

</details>

In your unit tests suite, prepare a Pact Provider Verification unit test:

1. Start your local Provider service
2. Optionally, instrument your API with ability to configure [provider states](https://github.com/pact-foundation/pact-provider-verifier/)
3. Run the Provider side verification step

To dynamically retrieve pacts from a Pact Broker for a provider with token authentication, instantiate a `PactBroker` object with your configuration:

```swift
// The provider being verified
let provider = ProviderVerifier.Provider(port: 8080)

// The Pact broker configuration
let pactBroker = PactBroker(
  url: URL(string: "https://broker.url/")!,
  auth: .token("auth-token"),
  providerName: "Some API Service"
)

// Verification options
let options = ProviderVerifier.Options(
  provider: provider,
  pactsSource: .broker(pactBroker)
)

ProviderVerifier().verify(options: options) {
  // do something (eg: shutdown the provider)
}
```

To validate Pacts from local folders or specific Pact files use the desired case.

<details><summary>Examples</summary>



```swift
// All Pact files from a directory example
ProviderVerifier()
  .verify(options: ProviderVerifier.Options(
    provider: provider,
    pactsSource: .directories(["/absolute/path/to/directory/containing/pact/files/"])
  ),
  completionBlock: {
    // do something
  }
)
```

```swift
// Only the specific Pact files
pactSource: .files(["/absolute/path/to/file/consumerName-providerName.json"])
```

```swift
// Only the specific Pact files at URL
pactSource: .urls([URL(string: "https://some.base.url/location/of/pact/consumerName-providerName.json")])
```

</details>

### Submitting verification results

To submit the verification results, provide `PactBroker.VerificationResults()` object to `pactBroker`.

<details><summary>Example</summary>

Set the provider version and optional provider version tags. See [version numbers](https://docs.pact.io/pact_broker/pacticipant_version_numbers) for best practices on Pact versioning

```swift
let pactBroker = PactBroker(
  url: URL(string: "https://broker.url/")!,
  auth: .token("auth-token"),
  providerName: "Some API Service",
  publishResults: PactBroker.VerificationResults(
    providerVersion: "v0.0.4-\(ProcessInfo.processInfo.environment["GITHUB_SHA"])",
    providerTags: ["\(ProcessInfo.processInfo.environment["GITHUB_REF"])"]
  )
)
```

</details>

## Matching

In addition to verbatim value matching, you can use a set of useful matching objects that can increase expressiveness and reduce brittle test cases.

See [Wiki page about Matchers][matchers] for a list of matchers `PactSwift` implements and their basic usage.

Or peek into [/Sources/Matchers/][pact-swift-matchers].

## Example Generators

In addition to matching, you can use a set of example generators that generate random values each time you run your tests.

In some cases, dates and times may need to be relative to the current date and time, and some things like tokens may have a very short life span.

Example generators help you generate random values and define the rules around them.

See [Wiki page about Example Generators][example-generators] for a list of example generators `PactSwift` implements and their basic usage.

Or peek into [/Sources/ExampleGenerators/][pact-swift-example-generators].

## Verifying your client against the service you are integrating with

If you set the `PACT_OUTPUT_DIR` environment variable, your Xcode setup is correct and your tests successfully run, then you should see the generated Pact files in:
`$(PACT_OUTPUT_DIR)/_consumer_name_-_provider_name_.json`.

Publish your generated Pact file(s) to your [Pact Broker][pact-broker] or a hosted service, so that your _API-provider_ team can always retrieve them from one location, even when pacts change. Normally you do this regularly in you CI step/s.

See how you can use simple [Pact Broker Client][pact-broker-client] in your terminal (CI/CD) to upload and tag your Pact files. And most importantly check if you can [safely deploy][can-i-deploy] a new version of your app.

## Objective-C support

PactSwift can be used in your Objective-C project with a couple of limitations, e.g. initializers with multiple optional arguments are limited to only one or two available initializers. See [Demo projects repository][demo-projects] for more examples.

```swift
_mockService = [[PFMockService alloc] initWithConsumer: @"Consumer-app"
                                              provider: @"Provider-server"
                                      transferProtocol: TransferProtocolStandard];
```

`PF` stands for Pact Foundation.

## Demo projects

[![PactSwift - Consumer](https://github.com/surpher/pact-swift-examples/actions/workflows/test_projects.yml/badge.svg)](https://github.com/surpher/pact-swift-examples/actions/workflows/test_projects.yml)
[![PactSwift - Consumer (macOS-10.15)](https://github.com/surpher/pact-swift-examples/actions/workflows/test_projects-macOS10_15.yml/badge.svg)](https://github.com/surpher/pact-swift-examples/actions/workflows/test_projects-macOS10_15.yml)
[![PactSwift - Provider](https://github.com/surpher/pact-swift-examples/actions/workflows/verify_provider.yml/badge.svg)](https://github.com/surpher/pact-swift-examples/actions/workflows/verify_provider.yml)

See [pact-swift-examples][demo-projects] for more examples of how to use `PactSwift`.

## Contributing

See:

- [CODE_OF_CONDUCT.md][code-of-conduct]
- [CONTRIBUTING.md][contributing]

## Acknowledgements

This project takes inspiration from [pact-consumer-swift](https://github.com/DiUS/pact-consumer-swift) and pull request [Feature/native wrapper PR](https://github.com/DiUS/pact-consumer-swift/pull/50).

Logo and branding images provided by [@cjmlgrto](https://github.com/cjmlgrto).

[action-default]: https://github.com/surpher/PactSwift/actions?query=workflow%3A%22Test+-+Xcode+%28default%29%22
[action-xcode11.5-beta]: https://github.com/surpher/PactSwift/actions?query=workflow%3A%22Test+-+Xcode+%2811.5-beta%29%22
[can-i-deploy]: https://docs.pact.io/pact_broker/can_i_deploy
[carthage_script]: ./Scripts/carthage
[code-of-conduct]: ./CODE_OF_CONDUCT.md
[codecov-io]: https://codecov.io/gh/surpher/PactSwift
[contributing]: ./CONTRIBUTING.md
[demo-projects]: https://github.com/surpher/pact-swift-examples
[example-generators]: https://github.com/surpher/PactSwift/wiki/Example-generators

[github-issues-52]: https://github.com/surpher/PactSwift/issues/52
[issues]: https://github.com/surpher/PactSwift/issues
[license]: LICENSE.md
[matchers]: https://github.com/surpher/pact-swift/wiki/Matchers
[pacticipant]: https://docs.pact.io/pact_broker/advanced_topics/pacticipant/
[pact-broker]: https://docs.pact.io/pact_broker
[pact-broker-client]: https://github.com/pact-foundation/pact_broker-client
[pact-consumer-swift]: https://github.com/dius/pact-consumer-swift
[pactswift-spec2]: https://github.com/surpher/PactSwift_spec2
[pact-docs]: https://docs.pact.io
[pact-reference-rust]: https://github.com/pact-foundation/pact-reference
[pact-slack]: http://slack.pact.io
[pact-specification-v3]: https://github.com/pact-foundation/pact-specification/tree/version-3
[pact-specification-v2]: https://github.com/pact-foundation/pact-specification/tree/version-2
[pact-swift-example-generators]: https://github.com/surpher/PactSwift/tree/main/Sources/ExampleGenerators
[pact-swift-matchers]: https://github.com/surpher/PactSwift/tree/main/Sources/Matchers
[pact-twitter]: http://twitter.com/pact_up
[releases]: https://github.com/surpher/PactSwift/releases
[rust-lang-installation]: https://www.rust-lang.org/tools/install
[slack-channel]: https://pact-foundation.slack.com/archives/C9VBGNT4K

[pact-swift-examples-workflow]: https://github.com/surpher/pact-swift-examples/actions/workflows/test_projects.yml

[upgrading]: https://github.com/surpher/PactSwift/wiki/Upgrading

[carthage-issue-3019-1]: https://github.com/Carthage/Carthage/issues/3019#issuecomment-665136323
[carthage-issue-3019-2]: https://github.com/Carthage/Carthage/issues/3019#issuecomment-734415287
[carthage-issue-3201]: https://github.com/Carthage/Carthage/issues/3201
