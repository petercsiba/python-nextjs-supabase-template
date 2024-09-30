# Template

## Stack
* Python w/ FastAPI
* Supabase for PostgresSQL
* `supawee` to connect Python to Supabase


### Setup Python
```shell
# in the template-backend directory
pyenv virtualenv 3.12.2 template-project
pyenv activate template-project
make requirements

# for IDE like intelliJ, you would need to setup the VirtualEnv
echo "$VIRTUAL_ENV/python"
# Sth like /Users/petercsiba/.pyenv/versions/3.12.2/envs/plugin-intelligence/python
# Copy this into PyCharm -> Settings -> ... -> Python Interpreter -> Add Local Interpreter
```

### Setup Git
```shell
# setup pre-commit checks
pre-commit install --config .pre-commit-config.yaml
pre-commit run --all-files  # check if works
```

If you using PyCharm like me make sure to update your project interpreter to the above virtualenv.


### Setup Supabase
Pretty much a relevant summary of https://supabase.com/docs/guides/getting-started/local-development
The configuration will live in `backend/supabase` (part of .gitignore).

```shell
supabase init
# PLEASE do NOT store your DB password here - keep using chrome or Doppler.
supabase link --project-ref template-supase-project
# Get remote migrations
supabase db remote commit
# Start bunch of stuff (you will need Docker daemon / Colima start);
supabase start
```

### Setup Docker (Colima)
I personally use `colima` over Docker Desktop (which kills battery)
For `colima` you need to install Docker `buildx` manually:
```shell
ARCH=arm64 # change to 'amd64' for non M[12]
VERSION=v0.10.4
curl -LO https://github.com/docker/buildx/releases/download/${VERSION}/buildx-${VERSION}.darwin-${ARCH}
mkdir -p ~/.docker/cli-plugins
mv buildx-${VERSION}.darwin-${ARCH} ~/.docker/cli-plugins/docker-buildx
chmod +x ~/.docker/cli-plugins/docker-buildx
docker buildx version # verify installation
```
https://dev.to/maxtacu/cross-platform-container-images-with-buildx-and-colima-4ibj
It somehow also worked better with x-platform local builds


## Development Workflow

### Running Tests
```shell
# For whatever reason, the tests are not running with the correct PYTHONPATH :/
# One day debug: https://chat.openai.com/share/1c47dfca-be7b-46f2-ad95-081d41346b18
PYTHONPATH=/Users/petercsiba/code/plugin-intelligence/backend pytest
```

### Committing Changes
We use `pre-commit` to run flake8, isort, black. Easiest to see results:
```shell
make lint-check  # optional for read-only mode
make lint  # actually modifies files
```
BEWARE: Sometimes `black` and `isort` disagree. Then:
* Run `pre-commit run --all-files` - this runs it clean without stash / unstash.
* If it passes, run `git status`. Likely the problem files aren't stage for commit (or have unstaged changes).
You might want to default to always do `-a` for `git commit`.

### Running Locally
```shell
uvicorn api.app:app --reload --port 8080
```


### Migrations: Create New Table
Most commands from https://supabase.com/docs/guides/getting-started/local-development#database-migrations

There are a few ways, the best feels like:
* Create a new table in Supabase UI: http://localhost:54323/project/default/editor
* Get the SQL table definition of it from the UI
* MAKE SURE it has RLS enabled, otherwise the new table has public access through PostREST (yeah :/).
  * Easiest with `ALTER TABLE your_table_name ENABLE ROW LEVEL SECURITY;` (alternatively with their UI)
  * Note that if the RLS policy is empty, then backend can still query it either via Service KEY or directly DB password.
* Add non-trivial stuff (like multicolumn indexes)
```shell
# Navigate to ./backend
supabase migration new your_migration_name
# and copy paste the SQL migration to definition to the generated file

# generate new python models (requires local supabase running)
supawee supabase/models/base.py

# apply migrations - weird name i know, this takes quite long :/
# supabase db reset # OG way
supabase migration up

# (optional): To take a snapshot of your current local data, you want to dump your DB data into `seed.sql`
export PGPASSWORD=postgres; pg_dump -h localhost -p 54322 -U postgres -d postgres --schema=public -F p --inserts > supabase/seed.sql

# Make sure the new schema works

# Then push migrations to be applied in prod (the Peewee models would mostly work^TM)
supabase db push
```