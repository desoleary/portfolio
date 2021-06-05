# Behaviour Driven Development (BDD) framework

## Preface
    - Greatly simplifies complex automation of API & UI Integration use-case    
    - Customized to support running [`Behat` framework](https://docs.behat.org/en/latest/) inside of the [`Dusk` runner](https://laravel.com/docs/8.x/dusk#managing-chromedriver-installations)
    - Use one feature file inside each runner 
    - Promotes extensive re-use throughout
    - Minimizes boilerplate involved in supporting new use-cases using convention over configuartion and dynamic handling of data techniques where possible.
    - Allows you to focus on writing concise use-case scenarios with a minimal amount of custom code
    - Can be used to safe-guard against introducing bugs via major refactoring and for example introducing a new framework

## Introduction

- Use the excellent [`Behat` framework](https://docs.behat.org/en/latest/) inside of dusk and in theory any other integration testing framework
- Concise Gherkin syntax to bring down the number of steps to fully test a feature
- Dynamically fill's in form and API data based on a single request object which is populated by given argument data passed in by each step
- Navigation is route based with simple identifier's E.g. "organization edit" page for "Organization #1" might goto /organizations/1/edit
- Having the test cases being called based on written scenarios can speed up development of otherwise tiresome and repetitive spec files
- As much boilerplate is extracted away in order to speed up development of test cases and make life easier as a developer and QA.
- Supports Audit Trail assertions especially important when dealing with financials

## Technical Information

#### Feature parser
> responsible for parsing a given feature file and provides useful methods

- Makes use of `Behat's` Gherkin Parser in order to transform a feature file into a form that provides some of the following:
- Ability to fetch background steps that will run prior to each scenario
- Ability to fetch all scenario titles and run each via [PHPUnit Data Provider](https://phpunit.readthedocs.io/en/9.5/writing-tests-for-phpunit.html#data-providers) feature   
- Ignore steps based on Gherkin [tags](http://docs.eggplantsoftware.com/ePF/using/epf-advanced-gherkin.htm#tags) E.g. `@UI_ONLY`
  

#### Scenario runner
> Adds the ability to run `Behat's` runner inside of `dusk` runner

- custom behat search engine in order to support filtering of feature files/scenarios
- includes a behat runtime handler which essentially `bootstraps behat` so it can be run inside another framework E.g. `dusk` without needing to run behat via their command line tool
- runs all scenarios in scope based on given feature file

#### Request Builder
> Responsible for building up request params / form inputs based on given arguments processed
- Supports adding, updating and deleting model entries via background and scenario steps
    - each persisted model gets stored into a dedicated repository object
- Supports building up request object that is used to call api endpoint or fill in form 

#### Form Control
> Responsible for filling and controlling forms / api params

- Automatically populates forms based on the likes of a given behat table argument (table nodes)
- In order to make this happen you simply need to have a consistent way of looking up each input E.g. label or `CSS Selector`

#### Table Control
> Responsible for controlling actions on table components

- Knows how to call actions E.g. delete, edit etc.
- In order to make this happen you simply need to have a consistent way of looking up each row E.g. "Organization #1"

#### Request Builder Factory
> Responsible for populating the main request builder the holds a reference to all model entries created as part of Background and Scenario steps

- Delegates to each model repository that handles individual CRUD operations and associated audit trail entries where applicable via [Laravel's Auditing Framework](http://www.laravel-auditing.com/)
- `model_reference` to refer to the actual [`Eloquent Model`](https://laravel.com/docs/8.x/eloquent#eloquent-model-conventions) type
  `reference` to refer to a specific database entry

#### Model Repository
> Holds reference to all database entries for a given model type and is responsible for all CRUD operations 

  - Each Database Eloquent Model Type has a separate repository object and associated factory that knows how to operate on a given [`Eloquent Model`](https://laravel.com/docs/8.x/eloquent#eloquent-model-conventions) type
     - Holds reference to all models of that type which can be referenced by a unique `reference`
     - Each model entry is built with defaults that can be overridden inside each step
       - Resembles the likes of Rails [`factory_bot`](https://github.com/thoughtbot/factory_bot) and [`Fixtures`](https://api.rubyonrails.org/v3.1/classes/ActiveRecord/Fixtures.html) in constructing useful and meaningful default values 
       - The importance of this is to ensure that each background and scenario step needs only to provide the minimum amount of data to support an individual use-case via the excellent and [sadly sunsetted](https://marmelab.com/blog/2020/10/21/sunsetting-faker.html) [Faker](https://github.com/fzaninotto/Faker) lib.
  - Simply refer to the `model_reference` E.g. `organization` with a unique `reference` E.g. `Organization #1` then repository object will know what to do based on given event E.g. "delete"
  - Handles all CRUD operations via `Laravel's` Eloquent Model in a generic and dynamic fashion using Eloquent Model column information
  - Ensures all data is sync'd to avoid un-necessary hits to the DB   

#### Model Attributes Mapper
> Responsible for mapping from and to Eloquent Model persisted layer

## Feature Files

#### Background Steps

- Adds the ability to create table entries with a given unique `reference` which can be referenced in other steps 
  - Minimum amount of data entry needed via useful defaults
- Navigation is kept as simple as possible and driven in a dynamic fashion via named routes

#### Scenario Steps
- Added the ability for the gherkin feature files to used inside of Larval's `dusk` browser automation framework via behat's context that allows for methods to be called based on an associated step 
  - E.g. `I am logged in as a user` would automatically fill in the login page
    `And I am on the update user page for "User #1"` would then describe where to navigate as a logged in user
    
- Single source of truth controlled via a request class
    - Regardless of the runner I keep a store of all background entries and
      As each scenario is run I have a class that contains references to each of the background table entries

- By default, each scenario can be run against the default `behat` runner or against `dusk` browser automation framework

###### Generic Sample Feature file

```gherkin
Background:
    Given an "organizations" table with the following information:
        | reference       | name  |
        | Organization #1 | Org A |
        | Organization #2 | Org B |
    Given an "users" table with the following information:
        | reference | email                         | organization_reference |
        | User #1   | desmond.oleary@kickbooster.me | Organization #1        |

    And I am logged in as a user
    And I am on the update user page for "User #1"
    
    @KCKB-699 # relates to a jira ticket that holds these generated scenarios
    Scenario: User details can be updated
    When I fill in the following inputs for "Organization #1":
        | organization_reference | email          |
        | Organization #2        | user@email.com |
    And I attempt to update the user
    Then the user should be updated
    And an audit entry should be created

    @KCKB-699
    Scenario: Organization is required
    When I fill in "organization" with "   " for "User #1"
    And I attempt to update the user
    Then an error message "This field is required." should be shown under the "organization_id" field

    @KCKB-700 @API_ONLY
    Scenario: Organization must exist
    When I fill in "organization" with "Org Z" for "User #1"
    And I attempt to update the user
    Then an error message "No matching organization id found." should be shown under the "organization_id" field

    @KCKB-701 # underlying I would have a fixture file with the name `file02.jpg`
    Scenario: Files can be attached
    When I add the following "files":
    | model_reference | file        |
    | User # 1        | file02.jpg  |
    And I attempt to update the transaction
    Then the transaction should be updated
    And an audit entry should be created
```

