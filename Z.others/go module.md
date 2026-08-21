go module

```go
module app-2

replace github.com/epmpub/strutils => ./strutils
replace github.com/epmpub/netutils => ./netutils
replace github.com/epmpub/person => ./person

go 1.26.5

require github.com/epmpub/strutils v0.0.0-00010101000000-000000000000
require github.com/epmpub/netutils v0.0.0-00010101000000-000000000000
require github.com/epmpub/person v0.0.0-00010101000000-000000000000

```

