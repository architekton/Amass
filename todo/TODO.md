TODO
====

Decouple / Refactor
--------

1. Make each service usable individually.

    * Separate services from enumeration to be able to construct the below example:

        ```go
        import "github.com/OWASP/Amass/amass/services"

        func main() {
        	req := NewRequestFromSubdomain(subdomain)
        	srv := NewAlterationService()
        	srv.Start()
        	srv.SendRequest(&req)
        	for sd := range srv.OutputChan() {
        	 	//code
        	}
        	// call srv.Stop() when the service is no longer active
        	// srv.Stop() also closes the output channel
        }
        ```

        or:

        ```go
        import "github.com/OWASP/Amass/amass/services"

        func output_callback(req *Request) {
        	// code
        }

        func main() {
        	req := NewRequestFromSubdomain(subdomain)
        	srv := NewAlterationService(output_callback)
        	srv.Start()
        	srv.SendRequest(&req)
        	// call srv.Stop() when the service is no longer active
        }
        ```

    *  Partial separation of services internally(every service can be integration tested) in order to produce a more configurable enumeration struct for example:

        ```go
        // Imports ommited

        func main() {
        	// Seed the default pseudo-random number generator
        	rand.Seed(time.Now().UTC().UnixNano())

        	// enum := amass.NewNullEnumeration() // creates an enumeration without services which does nothing on enum.Start()
        	// enum.AddService(ALTERATION)
        	// enum.AddService(DNS)
        	// enum.AddService(SOURCE_GOOGLE) // highly customisable
        	enum := amass.NewEnumeration()  // same as before so doesn't break anything
        	go func() {
        		for result := range enum.Output {
        			fmt.Println(result.Name)
        		}
        	}()
        	// Setup the most basic amass configuration
        	enum.Config.AddDomain("example.com")
        	enum.Start()
        }
        ```

3. <https://github.com/OWASP/Amass/blob/753d8cb4b58f76c1b597f9e88b4cdd25d16786de/amass/amass.go#L483> cleanName should be moved to util/

4. <https://github.com/OWASP/Amass/blob/753d8cb4b58f76c1b597f9e88b4cdd25d16786de/amass/amass.go#L52> Request tag types should be moved to the relevant request.go

5. Elimination of most magic strings by defining new constants in service.go. One section for data sources and one for the other service names.

Test / Bench
------------

1. All non-network functions are unit tested, mainly important utility functions that are used everywhere.
2. Passive data sources are integration tested as an individual service.
3. Alteration is unit tested and integration tested.
4. Benchmakrs are written for all widely used utility functions
5. Some services such as alteration and brute should have benchmarks. While this is a network tool and the speed of non-network functions is usually irrelevant, if the services are separated then amass can enumerate large files using standalone brute/alteration in order to use some other dns tool for example.

Questions / Suggestions
------------

1. Should we add a mutex to the request to prevent data races? There is currently one potentially harmless one detected by ` go run -race ./cmd/amass/main.go -active -d google.com`
2. Should **ALL** services get their input from SendRequest from the enumeration in order to start rather than from the config in order to remain consistent?
3. Should we add a new struct to the request called RequestStats in order to add internal data e.g the route the request took through the services, which initial request the current one derived from, how many copies were made, etc.
4. Should we define a clear point where sanitisation is performed e.g in alteration.go is correctRecordTypes necessary since the alteration depends only on `req.Name` and `req.Domain`, same with brute.go. Potentially:
    * All config options are checked in amass.go
    * A normally initialised service has no filters (except the necessary ones).
    * All sanitisation is performed within the service using a custom filters. E.g define a method `AddFilter(func(req *Request) *Request)` which uses function composition to compose multiple filters into a filter that will be ran
        before the actual heavy ops are performed. The filters may be defined in amass/filters.go, that way we can separate filters. The filter returns nil if the request didn't pass through.

Keep in mind I haven't read the complete source code (e.g graph and anything related to visualisation)
