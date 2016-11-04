all: up

up:
	@docker-compose up -d

clean:
	@docker-compose down

test-dns-internal:

	@echo "Test c1 -> c2 pings"

	# container one can reach container two using the service name
	@docker exec -t c1.gbifsweden.se ping -c 1 c2

	# container one can reach container two using the "container_name"
	@docker exec -t c1.gbifsweden.se ping -c 1 c2.gbifsweden.se

	# container one has its hostname set to the same as "container_name"
	@docker exec -t c1.gbifsweden.se hostname

	# but container two has not
	@docker exec -t c2.gbifsweden.se hostname

test-dns-external:

	# plain lookups
	@nslookup c1
	@dig c1
	@ping -c 1 c1

	# reverse lookup
	@dig -x `dig c1 +short`

	# fqdn lookups
	@dig gbifsweden.se
	@dig c1.gbifsweden.se
	@dig c1.gbifsweden.se.dnstestdocker_default

	# browse
	@firefox gbifsweden.se/
	@firefox c3/

	# does /etc/nsswitch.conf work?
	@getent hosts gbifsweden.se

test-dns-debug:

	@docker exec -it gbifsweden.se bash

