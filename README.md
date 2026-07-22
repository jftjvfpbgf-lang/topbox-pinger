# topbox-pinger

Scheduled pings for TopBox background jobs (survey dispatch every 5 minutes,
care tasks hourly). Lives in its own public repo because public-repo Actions
minutes are free; the target URL and auth secret are repository secrets —
nothing sensitive is in this code.
