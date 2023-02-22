# Canva
This is a readme for your new Budibase plugin.

# Description
Canva de cadastre

Find out more about [Budibase](https://github.com/Budibase/budibase).

## Instructions

To build your new  plugin run the following in your Budibase CLI:
```
budi plugins --build
```

You can also re-build everytime you make a change to your plugin with the command:
```
budi plugins --watch
```

## Make the plug in functionnal
#### You have to create a database with these column names:

| nom  | diminutif | role | image      |
|------|-----------|------|------------|
| text | text      | text | attachment |

#### Put the component in a dataProvider, then this dataProvider in a form.
#### Select the dataSourceS3 you want to use in ``DATASOURCE`` option
#### Select your dataProvider in ``INFOS POINTS`` option
#### Select your database in dataProvider ``Data`` option
#### Sort Column ``diminutif`` ascending to be sure ``aucune sélection`` row is first
