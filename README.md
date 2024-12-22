# golang-sample-with-nrpgx5

This repository is for demo how to use nrpgx5 for trace postgresql

## setup postgresql database

1. use docker compose with follow setup
```yaml
services:
  postgres:
    restart: always
    image: postgres:16
    container_name: postgres_docker_instance
    volumes:
      - ${HOST_DIR}:/var/lib/postgresql/data
    expose:
      - 5432
    ports:
      - ${POSTGRES_PORT}:5432
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    logging:
      driver: "json-file"
      options:
        max-size: "1k"
        max-file: "3"
```

2. run up postgresql instance

```shell
docker compose up -d postgres
```

## implement instructment with nrpgx5

```golang
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/jackc/pgx/v5"
	"github.com/leetcode-golang-classroom/golang-sample-with-nrpgx5/internal/config"
	"github.com/newrelic/go-agent/v3/integrations/nrpgx5"
	"github.com/newrelic/go-agent/v3/newrelic"
)

func main() {
	cfg, err := pgx.ParseConfig(config.AppConfig.DBURL)
	if err != nil {
		panic(err)
	}

	cfg.Tracer = nrpgx5.NewTracer(nrpgx5.WithQueryParameters(true))
	conn, err := pgx.ConnectConfig(context.Background(), cfg)
	if err != nil {
		panic(err)
	}

	app, err := newrelic.NewApplication(
		newrelic.ConfigAppName(config.AppConfig.AppName),
		newrelic.ConfigLicense(config.AppConfig.NewRelicLicenseKey),
		// newrelic.ConfigDebugLogger(os.Stdout),
	)
	if err != nil {
		panic(err)
	}
	//
	// N.B.: We do not recommend using app.WaitForConnection in production code.
	//
	app.WaitForConnection(5 * time.Second)
  // root transaction
	txn := app.StartTransaction("postgresQuery")

  // pass the root ctx into query
	ctx := newrelic.NewContext(context.Background(), txn)
	row := conn.QueryRow(ctx, "SELECT count(*) FROM pg_catalog.pg_tables")
	count := 0
	err = row.Scan(&count)
	if err != nil {
		log.Println(err)
	}

  // pass the root ctx into query
	var a, b int
	rows, _ := conn.Query(ctx, "select n, n*2 from generate_series(1, $1) n", 3)
	_, err = pgx.ForEachRow(rows, []any{&a, &b}, func() error {
		fmt.Printf("%v %v\n", a, b)
		return nil
	})
	if err != nil {
		panic(err)
	}
	txn.End()
	app.Shutdown(5 * time.Second)

	fmt.Println("number of entries in pg_catalog.pg_tables", count)
}
```