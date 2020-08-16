---
title: Advanced Rails Filter Parameter Logging 
date: "2020-08-16T12:40:32.169Z"
description: How to filter pesky sensitive parameters out of your logs using filter_parameter_logging option.
---

Have you ever wondered why you might see the `[FILTERED]` while debugging your rails application? Maybe you expect to see `[FILTERED]` for that new `super_secret_param` being sent to a newly created endpoint? Eitherway, this is the configuration option `filter_parameter_logging` at work. 
