# diego-capacity-monitor

[![codecov.io](https://codecov.io/github/FidelityInternational/diego-capacity-monitor/coverage.svg?branch=master)](https://codecov.io/github/FidelityInternational/diego-capacity-monitor?branch=master)
[![Go Report Card](https://goreportcard.com/badge/github.com/FidelityInternational/diego-capacity-monitor)](https://goreportcard.com/report/github.com/FidelityInternational/diego-capacity-monitor)
[![Build Status](https://travis-ci.org/FidelityInternational/diego-capacity-monitor.svg?branch=master)](https://travis-ci.org/FidelityInternational/diego-capacity-monitor)

`diego-capacity-monitor` is a [Cloud Foundry](https://www.cloudfoundry.org) deployable web application that subscribes to the [CF Firehose](https://docs.cloudfoundry.org/loggregator/architecture.html#firehose) to gather memory metrics about Diego cells. It then reports health states based a number of key metrics described below.

![Diego Monitor](diego-monitor.jpg "Diego Monitor")

### Operation

If there are no errors this application will return a json response similar to:

```
{
  healthy: true,
  message:"Everything is awesome!",
  details:[
    {
      index: 1,
      memory: 7000,
      low_memory: false
    },
    {
      index: 2,
      memory: 7000,
      low_memory: false
    }
  ]
  cellCount: 2,
  cellMemory: 10000,
  watermark: 1,
  requested_watermark: "1",
  totalFreeMemory: 14000,
  WatermarkMemoryPercent: 40
}
```

The following error messages and status can also be received:

- Its under a minute since the system was started
    - report.Message = "I'm still initialising, please be patient!"
    - report.Healthy = false
    - status = http.StatusExpectationFailed
- Invalid Watermark value supplied
    - reports.Message = "Error occurred while calculating cell count"
    - report.Healthy = false
    - status = http.StatusInternalServerError
- No metrics were found
    - report.Message = "I'm sorry Dave I can't show you any data"
    - report.Healthy = false
    - status = http.StatusGone
- The cellCount is not more than the watercount
    - report.Message = "The number of cells needs to exceed the watermark amount!"
    - report.Healthy = false
    - status = http.StatusExpectationFailed
- A third or more of the cells are under 2G memory free
    - report.Message = "At least a third of the cells are low on memory!"
    - report.Healthy = false
    - status = http.StatusExpectationFailed
- There is less memory free than the watermark amount
    - report.Message = "FATAL - There is not enough space to do an upgrade, add cells or reduce watermark!"
    - report.Healthy = false
    - status = http.StatusExpectationFailed
- During an upgrade there would be less than 20% memory free
    - report.Message = "The percentage of free memory will be too low during a migration!"
    - report.Healthy = false
    - status = http.StatusExpectationFailed

### Deployment

#### Watermark value:

This value can be supplied either as the number of cells to upgrade in parallel, or as a percentage.

#### Manual deployment

```
cf target -o <my_org> -s <my_space>
cf push --no-start
cf set-env diego-capacity-monitor CF_API_ENDPOINT <https://api.system.domain.cf>
cf set-env diego-capacity-monitor CF_USERNAME <CF_USERNAME_FOR_FIREHOSE_CONNECTION>
cf set-env diego-capacity-monitor CF_PASSWORD <CF_PASSWORD_FOR_FIREHOSE_CONNECTION>
cf set-env diego-capacity-monitor WATERMARK <optional, value will default to 1>
cf start diego-capacity-monitor
```

#### Automated zero-downtime deployment

```
CF_SYS_DOMAIN=system.example.cf.com \
CF_DEPLOY_USERNAME=cf_admin \
CF_DEPLOY_PASSWORD=123456789abcdef \
ORG_NAME=my_org \
SPACE_NAME=my_space \
CF_API_ENDPOINT=https://api.system.domain.cf \
CF_USERNAME=cf_firehose_username \
CF_PASSWORD=cf_firehose_password \
APP_NAME=my_diego_capacity_monitoring_app \
./deploy.sh
```

### Development

Currently, this repo should be manually cloned into `$GOPATH/src/github.com/FidelityInternational/diego-capacity-monitor`as the Godeps.json file has FidelityInternational github.com import path set (which will be used when we open source this project).

### Testing

#### Prereqs

```
brew install redis
go get github.com/EverythingMe/disposable-redis
go get github.com/onsi/gingko/ginkgo
```

#### Test

```
ginkgo -r -cover
```

#### Smoke Tests

```
APP_URL=<diego-capacity-monitor.apps.example.com> \
./smoke_test.sh
```
