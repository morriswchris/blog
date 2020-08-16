---
title: Advanced Rails Filter Parameter Logging 
date: "2019-09-08T22:40:32.169Z"
description: How to filter sensitive parameters out of your logs using filter_parameter_logging option, including parsing specific parameter formats.
---

Have you ever wondered why you might see the `[FILTERED]` while debugging your rails application? Maybe you expect to see `[FILTERED]` for that new `super_secret_param` being sent to a newly created endpoint? Eitherway, this is the configuration option `filter_parameter_logging` at work. 
