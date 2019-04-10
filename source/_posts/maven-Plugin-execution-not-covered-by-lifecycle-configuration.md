title: maven-Plugin execution not covered by lifecycle configuration
author: Xiang Chuang
tags:
  - maven
categories:
  - 爱学爱问
date: 2017-07-20 12:30:00
---
description:Project build lifecycle mapping can be configured in a project’s pom.xml, contributed by Eclipse plugins, or defaulted to the commonly used Maven plugins shipped with m2e. We call these “lifecycle mapping metadata sources”. m2e will create error marker like below for all plugin executions that do not have lifecycle mapping in any of the mapping metadata sources.

M2Eclipse matches plugin executions to actions using combination of plugin groupId, artifactId, version range and goal. There are three basic actions that M2Eclipse can be instructed to do with a plugin execution – ignore, execute and delegate to a project configurator(recommended).