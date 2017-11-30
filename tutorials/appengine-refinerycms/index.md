This tutorial shows how to create and configure [RefineryCMS](http://www.refinerycms.com/)
to run on Google App Engine flexible environment using
[Google Cloud SQL](https://cloud.google.com/sql) and
[Google Cloud Storage](https://cloud.google.com/storage).

## Objectives

* Create a new RefineryCMS project
* Create a Google Cloud Storage bucket
* Create a new Cloud SQL for PostgreSQL instance
* Prepare deployment configuration
* Deploy to Google App Engine flexible environment
* Run database migrations
* Access the deployed RefineryCMS project

## Before you begin

You'll need the following:

* A Google Cloud Platform (GCP) project. You can use an existing project or
  click the button to create a new project.
* [Ruby 2.4.0+ installed](https://www.ruby-lang.org/en/documentation/installation/)
* Rails 5.0+ gem installed
* RefineryCMS 4.0+ gem installed
* [Google Cloud SDK installed](https://cloud.google.com/sdk/downloads)

## Costs

This tutorial uses billable components of Cloud Platform including:

* Google App Engine flexible environment
* Google Cloud Cloud SQL
* Google Cloud Storage

Use the [pricing calculator](https://cloud.google.com/products/calculator)
to generate a cost estimate based on your projected usage. New GCP users might
be eligible for a [free trial](https://cloud.google.com/free-trial).

## Create a new RefineryCMS project

RefineryCMS can be use to manage content for your website. Next create a new
RefineryCMS project using the `refinerycms` gem:


    refinery website_name

You have a new directory named for example `website_name` that contains the
generated RefineryCMS project. Change directories into the generated project
using the following command:

    cd website_name

If you are familiar with Ruby on Rails, you will notice that RefineryCMS uses
Ruby on Rails at its core and adds additional funtionality to manage content.

You can test the generated RefineryCMS project using the following command:

    bin/rails server

This will start a server listening for incoming requests at
`http://localhost:3000`. You should see the following welcome page:

![local hello world](./images/local_hello_world.jpg)

## Create a Google Cloud Storage bucket

You will use Google Cloud Storage to host files and images for RefineryCMS.

Before configuring RefineryCMS, create a new bucket for named
`[YOUR-PROJECT-ID]-refinery-cms`  by using the following gsutil command:

    gsutil mb gs://[YOUR-PROJECT-ID]-refinery-cms

This will make a globally unique bucket named [YOUR-PROJECT-ID]-refinery-cms.

RefineryCMS uses Dragonfly to interact with cloud data stores. To configure
Dragonfly to use Google Cloud Storage configure the custom backend
configuration exposed by RefineryCMS.

    bundle add dragonfly-google_data_store

Next, we will update the file `config/initializers/refinery/core.rb` to include
the following:

    # The following require statement to core.rb
    require "dragonfly/google_data_store"

    # This line is in core.rb. It’s near the top.
    Refinery::Core.configure do |config|

      # The following two lines are commented and should be set
      # with the following values.
      config.dragonfly_custom_backend_class = "Dragonfly::GoogleDataStore"

      config.dragonfly_custom_backend_opts = {
          project: ENV["GOOGLE_CLOUD_PROJECT"],
          bucket:  "#{ENV['GOOGLE_CLOUD_PROJECT']}-refinery-cms"
      }
      # The rest of this file was truncated.
    end

Your RefineryCMS project is now ready to use Google Cloud Storage to store file
and images.

## Create a new Cloud SQL for PostgreSQL instance

Cloud SQL for PostgreSQL is a fully managed database service that makes it easy
to set up, maintain, manage, and administer your relational PostgreSQL databases
in the cloud. That means you can use Cloud SQL in a Rails app like any other
relational database.

First, create a Second Generation instance named for example
`refinery-cloudsql-instance`.

    gcloud beta sql instances create refinery-cloudsql-instance \
      --database-version POSTGRES_9_6 \
      --cpu=1 --memory=3840MiB --region=us-central1

The operation will take a few moments to complete. Learn more about
[creating instances](https://cloud.google.com/sql/docs/postgres/create-instance).
You should see the following:

    NAME                     REGION       TIER              STATUS
    rails-cloudsql-instance  us-central1  db-custom         RUNNABLE

Next, create a new database in the new instance named for example
`refinery_database_production`.

    gcloud sql databases create refinery_database_production \
      --instance=refinery-cloudsql-instance

You should see the following:

    Creating Cloud SQL database...done.
    Created database [cat_list_production].
    instance: refinery-cloudsql-instance
    name: refinery_database_production
    Project: YOUR_PROJECT_ID

Finally, set a root user password for the instance:

    gcloud sql users set-password postgres no-host \
      --instance=refinery-cloudsql-instance \
      --password=some-password

The Cloud SQL for PostgreSQL instance will only be accessible to the
RefineryCMS project when deployed to Google App Engine flexible environment.

Next update the Refinery CMS app to use CloudSQL in production.

Add gems `pg` and `appengine` to the `Gemfile` file using the following
commands:

    bundle add pg
    bundle add appengine

To connect to your Google CloudSQL production database you will need the
**Instance Connection Name**. Get the Instance Connection Name using the
following command:

    gcloud sql instances describe refinery-cloudsql-instance | grep connectionName

The Instance Connection Name will be the name of a UNIX_SOCKET the app will use
to connect to the CloudSQL instance refinery-cloudsql-instance when deployed
to App Engine flexible environment.

Open the file config/database.yml and update the production environment settings
to the following.

    production:
      encoding: unicode
      adapter: postgresql
      pool: 5
      timeout: 5000
      username: "postgres"
      password: "some-password"
      database: "refinery_database_production"
      host:   "/cloudsql/[YOUR_INSTANCE_CONNECTION_NAME]"

Save and close the file.


## Prepare deployment configuration

App Engine Flexible environment uses a file called app.yaml to describe an
application's deployment configuration. If this file is not present, the gcloud
tool will try to guess the deployment configuration. However, it is a good idea
to provide this file because Rails requires a secret key in production.

A secret key is used to protect user session data. You can generate a secret
key by using:

    bin/rails secret

Copy the secret key that’s generated to your clipboard because you will use it
in the next step.

Create a new file named `app.yaml` and add the following to the file:

    entrypoint: bundle exec rails server --port $PORT
    env: flex
    runtime: ruby

    env_variables:
      SECRET_KEY_BASE: [SECRET KEY]

    beta_settings:
      cloud_sql_instances: [YOUR_INSTANCE_CONNECTION_NAME]

Replace [SECRET_KEY] with the generated secret key and
[YOUR_INSTANCE_CONNECTION_NAME] with the CloudSQL Instance Connection Name used
in your `config/database.yml` file.

When the app is deployed, the environment variable SECRET_KEY_BASE in production
will be set to the secret key that was generated. The environment variable
`SECRET_KEY_BASE` is used in `config/secrets.yml`.

Lastly, precompile your RefineryCMS assets for production by using the following
command:

    RAILS_ENV=production bin/rails assets:precompile

## Deploy to Google App Engine flexible environment

Next you will deploy your app to App Engine which will be used for database
migrations and finally as the default serving app.

First, you need to create an App Engine instance by using:

    gcloud app create --region us-central

Learn more about [Regions and Zones](https://cloud.google.com/docs/geography-and-regions).

After this is enabled, you can deploy your app.

    gcloud app deploy

The deployment may take several minutes. This is because App Engine flexible
environment will automatically provision a Google Compute Engine virtual machine
for you behind the scenes, and then install the application, and start it.

## Run database migrations

The Appengine gem that was included into the RefineryCMS `Gemfile` is used to
run production database migrations when your application is deployed to Google
App Engine flexible environment.

The appengine gem provides the Rake task `appengine:exec` to run a command
against the most recent deployed version of your app in the production App
Engine flexible environment.

Ruby on Rails database migrations are used to update the schema of your database
without writing SQL.

Before you can use the gem to run database migrations grant access to the
`cloudbuild` service account.

Retrieve the project number by listing available projects by using the following
command:

    gcloud projects list

Copy the project number for your project. Use the following command to add
a new member to your project IAM policy for the role `roles/editor`.

    gcloud project add-iam-policy-binding [YOUR-PROJECT-ID] \
        --member=serviceAccount:[PROJECT-NUMBER]@cloudbuild.gserviceaccount.com \
        --role=roles/editor

Next you will perform a database migration for your production database
`refinery_database_production`.

Run the database migration in production using the following command:

    bin/rake appengine:exec -- bundle exec rails db:migrate

The `bundle exec rake appengine:exec` command migrates the production Cloud SQL
for PostgreSQL database `refinery_database_production`. You should see a similar
success output:

    ---------- EXECUTE COMMAND ----------
    bundle exec rake db:migrate
    Debuggee gcp:787021104993:8dae9032f8b02004 successfully registered
    == 20170804210744 CreateCats: migrating =======================================
    -- create_table(:cats)
    -> 0.0219s
    == 20170804210744 CreateCats: migrated (0.0220s)
    ==============================

    ---------- CLEANUP ----------


Finally, RefineryCMS requires seed data and can be generated in production
using the following command:

    bin/rake appengine:exec -- bundle exec rails db:seed


## Access the deployed RefineryCMS project

You should now be able to access your RefineryCMS project website by using the
following command or pointing your browser to
`https://YOUR-PROJECT-ID.appspot.com`:

    gcloud app browse


You should see the following page similar to your local test.

    [!remove-version]()



## Cleaning up

After you've finished this tutorial, you can clean up the resources you created
on Google Cloud Platform so you won't be billed for them in the future. The
following sections describe how to delete or turn off these resources.

## Deleting the project

The easiest way to eliminate billing is to delete the project you created for
the tutorial.

To delete the project:

1. In the Cloud Platform Console, go to the
   **[Projects](https://console.cloud.google.com/iam-admin/projects)** page.
1. Click the trashcan icon to the right of the project name.

**Warning**: Deleting a project has the following consequences:

If you used an existing project, you'll also delete any other work you've done
in the project. You can't reuse the project ID of a deleted project. If you
created a custom project ID that you plan to use in the future, you should
delete the resources inside the project instead. This ensures that URLs that use
the project ID, such as an appspot.com URL, remain avaiable.

### Deleting App Engine services

To delete an App Engine service:

1. In the Cloud Platform Console, go to the
   **[App Engine Services](https://console.cloud.google.com/appengine/services)** page.
1. Click the checkbox next to the service you wish to delete.
1. Click **Delete** at the top of the page to delete the service.

If you are trying to delete the *default* service, you cannot. Instead:

1. Click on the number of versions which will navigate you to the App Engine
   Versions page.
1. Select all the versions you wish to disable and click **Stop** at the top of
   the page. This will free all of the Google Compute Engine resources used for
   this App Engine service.

