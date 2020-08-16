---
title: Advanced Filter Parameter Logging in Rails
date: "2020-08-16T12:40:32.169Z"
description: How to filter pesky sensitive parameters out of your logs using filter_parameter_logging option.
---

Rails in a powerful framework, providing great _magical_ features with little to no configuration. For instance, if you were to turn your logging level to `info`, magically you can now not only see the requested URL for your controllers, but the parameters that were provided as well! Think about that last sentence again. All of your parameters will be logged ... in plain text to your logging provider. Suddenly this helpful configuration change has added a potential security attack vector to your application. Your logs will now contain any secrets being passed to your controllers. Thankfully rails has a not so straightforward configuration option we'll walk through together below: `config.filter_parameters `

## Setting up the config

The bare minimum configuration for filtering certain parameters from your logs is to set up an array of keys that should not be included in your logs. It will replace the value of the key with `[FILTERED]`. Let's look at what Rails provides us in terms of documentation:

> `config.filter_parameters` used for filtering out the parameters that you don't want shown in the logs, such as passwords or credit card numbers. It also filters out sensitive values of database columns when call #inspect on an Active Record object. By default, Rails filters out passwords by adding Rails.application.config.filter_parameters += [:password] in config/initializers/filter_parameter_logging.rb. Parameters filter works by partial matching regular expression.

This seems easy enough! By appending the keys you wish to be filtered from your parameters in a new `initializers` file, we can rest assured our logs don't contain any exposed secrets ... granted the parameters can be parsed. 

## Filter non-parsable params

Sometimes Rails has a hard time parsing a set of parameters to the correct format. Have you even setup a new webhook handler route, to discover the payload is of content `multipart/form-data`, but a form field is a stringified JSON? Sometimes external payloads aren;t configured properly, but still need to be parsed all the same. Thankfully `config.filter_parameters` supports `lambda` functions as a value in the Array! Setting up a lambda to do some extra parsing for us, we can filter out additional parameters as if they were parsed correctly the first time. We will set this up with two files, one to hold the lambda, and the existing `initializers` file to apply the configuration.

### app/helpers/filter_helper.rb

```
# frozen_string_literal: true

module FilterHelper
  # Configure sensitive parameters which will be filtered from the log file.
  ALWAYS_FILTERED_PARAMS = [
    :encrypted_token,
    :password,
    :code,
    :response_url,
    :token,
    :trigger_id,
    :sharedSecret,
    :installation_id,
    :state_hash,
    :SAMLResponse
  ]

  # Define the parameter keys which need to be parsed from a JSON string
  JSON_FILTER_PARAMS = ['payload']

  # Filter by a particular param key whose value is a JSON encoded string
  # based on our standard set of filtering parameters.
  FILTER_STRING_TO_JSON = lambda do |key, value|
    if JSON_FILTER_PARAMS.include?(key.to_s) && value.is_a?(String)
      json = JSON.parse(value)
      filtered = param_filter.filter(json)
      value.replace(filtered.to_json)
    end
  rescue JSON::ParserError
    Rails.logger.info("Parameter #{key} did not contain a JSON parsable value")
  end

  def self.param_filter
    ActionDispatch::Http::ParameterFilter.new(ALWAYS_FILTERED_PARAMS)
  end
end

```

In the above file we setup a few constants, `ALWAYS_FILTERED_PARAMS` to keep track of the parameters we wish to filter out (including sensitive parameters in our JSON string), `JSON_FILTER_PARAMS` to keep track of parameter keys which contain stringified JSON to parse, and `FILTER_STRING_TO_JSON` as our lambda to filter our values. Lastly we have `self.param_filter` which allows us to call `filter` on our parsed values, doing the necessary substitutions.

**Bonus**: Since we moved our lambda into a helper module, it can be unit tested in isolation to our configuration changes!

### config/initializers/filter_parameter_logging.rb

```
# frozen_string_literal: true

Rails.application.config.filter_parameters += FilterHelper::ALWAYS_FILTERED_PARAMS
Rails.application.config.filter_parameters << FilterHelper::FILTER_STRING_TO_JSON
```

Once we have our helper setup we can adjust how we use `config.filter_parameters`. We start by applying the array of symbols and strings to be filtered. Next we append to the array our lambda, which will do the additional filtering for us After a restart of your server, you should begin to see those pesky parameters being filtered from your logs!
