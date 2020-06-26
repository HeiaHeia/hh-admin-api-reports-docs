---
title: Admin API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell

toc_footers:
  - <a href='https://heiaheia.com'>HeiaHeia</a>

includes:
  - errors

search: true
---

# Introduction to HH Admin API

This is short API documentation for fetching HH reports data. Before taking the API in to use, make sure to contact our technical team (support@heiaheia.com).

We have examples for all endpoints in shell, but you can easily use any programming language you want by making indentical requests.

If you need the whole admin API documentation, you can find it in [SwaggerHub](https://app.swaggerhub.com/apis/Hintsa/heia-heia_admin_api/1.0.1).

# Authentication

> To authorize, make the following request:

```shell
curl -XPOST -H "Content-Type: application/json" -d '{"grant_type": "magic_token", "email": "client@client.com", "magic_token": "{magic_token}", "client_id": "{client_id}", "client_secret": "{client_secret}"}' 'https://api.heiaheia.com/oauth/token' | json_pp
```

> The above command returns JSON structured like this:

```json
{
  "access_token": "123123",
  "scope": "read_only_admin_api",
  "created_at": 1521446718,
  "token_type": "bearer"
}
```

Most API calls require authentication and the first step is to get an access token. The access token created using the mechanism described here are valid for two weeks. The same call can be used to get a new access token and you can request new one before the previous one expires so one possible approach would be to implement a daily data synchronization where the daily sync starts by obtaining new access token, performs all sync operations for all organizations, and then discards the access token and gets a new one the next day.

The returned access token can only be used to perform read (HTTP GET) operations, which should be sufficient for implementing first phase of the integration. Any operations that would modify system state will fail with HTTP status code 403.

<aside class="notice">
You must replace <code>email, magic_token, client_id, client_secret</code> with the ones provided to you by the application team.
</aside>

# Organisations

## Listing organisations

```shell
curl -XGET 'https://api.heiaheia.com/admin/v1/documents/organisations?page=1&per_page=500&size_classes=50,10000&access_token=123' | json_pp
```

> The above command returns JSON structured like this:

```json
{
   "total": 500,
   "items" : [
      {
         "size_class" : 10000,
         "id" : 123,
         "name" : "Demo company 1",
         "multi_business_id": true,
         "multi_business_ids": [
           "123123-1",
           "123123-2"
        ]
      },
      {
         "size_class" : 10000,
         "id" : 1234,
         "name" : "Demo company 2",
         "multi_business_id": false
      },
    …
  ]
}

```

Once you’ve obtained the access token the next step is to list all organizations that may have interesting data. The organization listing API is paged and maximum page size is 500, which is less than the current number of applicable organizations so the request needs to be repeated with an increasing page number until all data has been received. The size_classes parameter should be used to filter out small organizations, which can never have interesting data. To get the first page of relevant organizations make a call like this described here.

At this point you can also see if an Organisation contains multiple business ids or not based on the attributes `multi_business_id` and `business_ids`.

### HTTP Request

`GET https://api.heiaheia.com/admin/v1/documents/organisations`

### Query Parameters

| Parameter    | Default | Description                                                |
| ------------ | ------- | ---------------------------------------------------------- |
| page         | 1       | Page number                                                |
| per_page     | true    | How many items per page                                    |
| size_classes |         | Filter organisations based on size classes (20, 50, 10000) |

<aside class="notice">
You will only see organisations available for your user.
</aside>

# Survey & usage data

## Getting organization specific data

After getting all the organization names and ids, next step is to get relevant data for each organization. There is currently no API that would allow mass access to multiple organizations’ data but making the necessary API calls for each organization one by one should complete in around 15 minutes with the current data set (without any concurrency) so the simplistic approach should be good enough for the foreseeable future, presuming synchronizing all data once a day is sufficient. Not doing operations concurrently is preferable from our point-of-view to avoid causing unnecessary load on our servers due to the background synchronization.

There are 3 calls that should be done for each organization and an additional fourth for organizations that have completed an organization survey since the last synchronization.

The first call is to get usage statistics for the organization, the second is to get “Test your energy levels” survey results, the third one is to get organization survey report metadata, and the fourth is getting actual report data for organizations that have completed a new organization survey.

The second call to get the“Test your energy levels” survey results could/should be skipped for organizations that don’t have sufficient number of activated HH users (many organizations only use the organization survey part of the platform) and it should only be called on dates when reporting period has changed (e.g. if results are reported for one month period this call should only be made on the first day of each month for the previous month).

## Usage stats for an individual organization

```shell
curl -XGET 'https://api.heiaheia.com/admin/v1/documents/organisations/123/stats?access_token=123123' | json_pp
```

> The above command returns JSON structured like this:

```json
{
  "usage": {
    "total_membership_count": 20,
    "hh_core_total_membership_count": 0,
    "hh_core_accepted_membership_count": 0,
    "app_use_count": 2,
    "app_monthly_use_count": 2
  }
}
```

Usage stats for an individual organization can be fetched with a call like described here.

The data returned by the usage stats API is current live values for each field. In order to display trends or historical values the caller must persist the date/time of the call along with the data so that it can compute the delta in the last month or whatever the reporting period is.

<aside class="notice">
You can filter the stats by business id by providing it as a parameter. In order for this to work, the business id has to be mapped to users inside the Organisation.
</aside>

### HTTP Request

`GET https://api.heiaheia.com/admin/v1/documents/organisations/<ID>/stats`

### URL Parameters

| Parameter   | Description                                                 |
| ----------- | ----------------------------------------------------------- |
| ID          | The ID of the organsation                                   |
| business_id | Filter report for specific business-id in this organisation |

## “Test your energy levels” survey data for an individual organization

```shell
curl -XGET 'https://api.heiaheia.com/admin/v1/documents/organisations/123/energy_levels?access_token=123123&start_date=2019-01-01&end_date=2019-02-28' | json_pp
```

> The above command returns JSON structured like this:

```json
{
  "organisation": {
    "id": 123,
    "name": "Bahringer Inc",
    "business_id": "123123-1",
    "size_class": 20,
    "su_access": false
  },
  "questions": [
    {
      // Questions listed
    }
  ],
  "teams": [
    {
      // Teams listed
  ],
  "instances": [
    {
      "minimum_answer_count": 7,
      "global": {
        // Global results
      },
      "organisation": {
        // Organisation results
      },
      "teams": {
        "3790": {
          // Individual team results
        }
      }
    }
  ]
}
```

The “Test your energy levels” survey results can be fetched for an individual organization with a call like described here.

The start_date and end_date parameters specify the (inclusive) date range from which to gather results. Even though it is possible to pass date ranges that are long in the past, it is recommended to only use this API to get results immediately after the desired date and store the results locally. The server computes the results dynamically based on current data so if users are removed from the organization, their teams change or any other similar changes take place the data for past dates will change, which is almost always undesired.

There's no point in fetching these for organisations where `hh_core_accepted_membership_count` < 5

The full response data format is too long to show here, so please refer to the API documentation at https://app.swaggerhub.com/apis/Hintsa/heia-heia_admin_api/1.0.1 for exact format.

<aside class="notice">
You can filter the stats by business id by providing it as a parameter. In order for this to work, the business id has to be mapped to users inside the Organisation.
</aside>

### HTTP Request

`GET https://api.heiaheia.com/admin/v1/documents/organisations/<ID>/energy_levels`

### URL Parameters

| Parameter   | Description                                                 |
| ----------- | ----------------------------------------------------------- |
| ID          | The ID of the organsation                                   |
| start_date  | Start of the selected date range                            |
| end_date    | End of the selected date range                              |
| business_id | Filter report for specific business-id in this organisation |

## Listing survey reports for an individual organization

```shell
curl -XGET 'https://api.heiaheia.com/admin/v1/documents/organisations/123/reports?access_token=123123&viewed=true' | json_pp
```

> The above command returns JSON structured like this:

```json
{
  "total": null,
  "items": [
    {
      "id": 708,
      "data_instance": {
        "id": "organisation_123_survey_key_tyovire_123",
        "type": "OrganisationSurveyReportDataInstance"
      },
      "viewed": true,
      "sub_report": false,
      "first_viewed_at": "2018-03-14T16:22:20+02:00",
      "generated_at": "2018-03-14T16:22:20+02:00",
      "scheduled_survey": {
        "id": 172,
        "started_at": "2019-05-07T11:08:37Z",
        "ends_on": "2019-05-14",
        "closed": true,
        "title": "Survey title here",
        "survey": {
          "id": 215,
          "title": {
            "en": "Työvire"
          }
        },
        "enabled_questions": [9, 10, 11, 12, 14, 27],
        "copy_questions": ["enps"]
      }
    },
    {
      "id": 709,
      "data_instance": {
        "id": "190faf097013a5ef2fccf3941c02e292de606743_2fd79f8b47a2b9c7d63b73276606587bc58a0263a02b0a47fdcd333ae66c",
        "type": "OrganisationSurveyReportDataInstance"
      },
      "viewed": true,
      "sub_report": true,
      "sub_report_business_id": "123123-1",
      "first_viewed_at": "2018-03-14T16:22:20+02:00",
      "generated_at": "2018-03-14T16:22:20+02:00",
      "scheduled_survey": {
        "id": 172,
        "started_at": "2019-05-07T11:08:37Z",
        "ends_on": "2019-05-14",
        "closed": true,
        "title": "Survey title here",
        "survey": {
          "id": 215,
          "title": {
            "en": "Työvire"
          }
        },
        "enabled_questions": [9, 10, 11, 12, 14, 27],
        "copy_questions": ["enps"]
      }
    }
  ]
}
```

The viewed=true parameter restricts results to reports that have been generated and viewed by the organization HR admin. The data for reports that have not been viewed is not accessible via the API so there’s no point in returning those. The most recent reports are returned first.

If the organisation has multiple business ids, this list will contain also all "sub reports". You can diffrentiate the sub reports from the main organisation report by the `sub_report` and `sub_report_business_id` fields.

<aside class="notice">
From this endpoint you are interested in the `data_instance.id` for the next request.
</aside>

### HTTP Request

`GET https://api.heiaheia.com/admin/v1/documents/organisations/<ID>/reports`

### URL Parameters

| Parameter | Description                                           |
| --------- | ----------------------------------------------------- |
| ID        | The ID of the organsation                             |
| viewed    | Restricts results to reports that have been generated |

## Getting survey data for an individual report

```shell
curl -XGET 'https://api.heiaheia.com/reports/v1/documents/reports/organisation_123_survey_key_tyovire_123/full_data' | json_pp
```

> The above command returns JSON structured like this:

```json
{
  "title": "Survey title",
  "sub_report": true,
  "report_business_id": "123123-1",
  "report_business_desc": "Description about sub company",
  "organisation": {
    "id": 265,
    "name": "Demo company 1",
    "business_id": "123123-0",
    "size_class": 20,
    "su_access": false,
    "total_membership_count": 116,
    "hh_core_total_membership_count": 0,
    "hh_core_accepted_membership_count": 0,
    "feature_tags": [],
    "multi_business_id": true,
    "company_code": "123123123123"
  },
  "survey": {
    // Survey info
  },
  "explicit_team_ids": null,
  "explicit_tag_ids": null,
  "teams": [
    // Teams listed here
  ],
  "tags": [
    // Tags listed here
  ],
  "questions": [
    // Questions listed here
  ],
  "question_categories": [
    // Question categories listed here
  ],
  "instances": [
    {
      "generated_at": "2019-05-21T11:10:59+00:00",
      "start_time": "2019-05-07T11:08:37Z",
      "ends_on": "2019-05-14",
      "minimum_answer_count": 5,
      "point_categories": {
        "minimum": 1,
        "maximum": 5,
        "step": 0.5
      },
      "global": {
        // Global results
      },
      "organisation": {
        // Organisaton results
      },
      "subgroup": null,
      "filtered_subgroup": null,
      "teams": {
        // Team results
      },
      "filtered_teams": null,
      "tags": {
        // Tag results
      },
      "open_questions": [
        // Open questions
      ],
      "similarities": [
        // Similarietes between teams and tags
      ]
    }
  ],
  "id": "190faf097013a5ef2fccf3941c02e292de606743_2fd79f8b47a2b9c7d63b73276606587bc58a0263a02b0a47fdcd333ae66c"
}
```

The actual report data is accessible using an API call like described here.

The full response data format is too long to show here, so please make the actual request to see all the data. Here we have listed the main contents of the report.

The result might contain multiple instances if the survey has been done multiple times. The latest result will be the first item in the `instances` array. You are probably interested in downloading & parsing it. The rest of the instances can used for calculating historical comparisons etc.

On the top level there can be fields `sub_report` and `report_business_id` if the report is a sub report for specific business id. This is same information you can find in the previous endpoint before making this request.

<aside class="notice">
Note that this call does not require authentication. Knowing the report data instance identifier (as returned by the earlier admin API call) is sufficient for accessing the report data. This call should only be made for reports that have not already been downloaded and stored on client side.
</aside>

### HTTP Request

`GET https://api.heiaheia.com/admin/v1/documents/reports/<ID>/full_data`

### URL Parameters

| Parameter | Description                 |
| --------- | --------------------------- |
| ID        | The ID of the data instance |
