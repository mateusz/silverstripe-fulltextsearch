# Solr connector for SilverStripe fulltextsearch module

## Introduction

This module provides a fulltextsearch module connector to Solr. 
It works with Solr in multi-core mode. It needs to be able to update Solr configuration files, and has modes for
doing this by direct file access (when Solr shares a server with SilverStripe) and by WebDAV (when it's on a different server).

See the helpful [Solr Tutorial](http://lucene.apache.org/fulltextsearch/api/doc-files/tutorial.html), for more on cores and querying.

## Requirements

Since Solr is Java based, it requires Java 1.5 or greater installed. 
It also requires a servlet container such as Tomcat, Jetty, or Resin.
Jetty is already packaged with the module.

See the official [Solr installation docs](http://wiki.apache.org/solr/SolrInstall)
for more information.

Note that these requirements are for the Solr server environment,
which doesn't have to be the same physical machine as the SilverStripe webhost.

## Installation

Configure Solr in file mode. The 'path' directory has to be writeable
by the user the Solr search server is started with (see below).

	// File: mysite/_config.php:
	<?php
	SearchUpdater::bind_manipulation_capture();
	Solr::configure_server(isset($solr_config) ? $solr_config : array(
		'host' => 'localhost',
		'indexstore' => array(
			'mode' => 'file',
			'path' => BASE_PATH . '/.solr'
		)
	));

Note: We recommend to put the `indexstore.path` directory outside of the webroot.
If you place it inside of the webroot (as shown in the example),
please ensure its contents are not accessible through the webserver.
This can be achieved by server configuration, or (in most configurations)
also by marking the folder as hidden via a "dot" prefix.

Create an index

	// File: mysite/code/MyIndex.php:
	<?php
	class MyIndex extends SolrIndex {
		function init() {
			$this->addClass('Page');
			$this->addAllFulltextFields();
		}
	}

Start the search server (via CLI, in a separate terminal window or background process)

	cd fulltextsearch/thirdparty/solr/server/
	java -jar start.jar

Initialize the configuration (via CLI)

	sake dev/tasks/Solr_configure

Based on the sample configuration above, this command will do the following:

- Create a <BASE_PATH>/.solr/MyIndex` folder
- Copy configuration files from `fulltextsearch/conf/extras/` to `<BASE_PATH>/.solr/MyIndex/conf`
- Generate a `schema.xml`, and place it it in `<BASE_PATH>/.solr/MyIndex/conf`

If you call the `Solr_configure` task with an existing index folder,
it will overwrite all files from their default locations, 
regenerate the `schema.xml`, and ask Solr to reload the configuration.

## Usage

After configuring Solr, you have the option to add your existing
content to its indices. Run the following command:

	sake dev/tasks/Solr_reindex

This will delete and rebuild all indices. Depending on your data,
this can take anywhere from minutes to hours.
Keep in mind that the normal mode of updating indices is
based on ORM manipulations of the underlying data.
For example, calling `$myPage->write()` will automatically
update the index entry for this record (and all its variants).

You can narrow down the operation with the following options:

 - `index`: PHP class name of an index
 - `class`: PHP model class to reindex
 - `start`: Offset (applies to matched records)
 - `variantstate`: JSON encoded string with state, e.g. '{"SearchVariantVersioned":"Stage"}'
 - `verbose`: Debug information
 - `keepindex`: If set to "true", won't clean existing index entries.

Note: The Solr indexes will be stored as binary files inside your SilverStripe project. 
You can also copy the `thirdparty/` solr directory somewhere else,
just set the `path` value in `mysite/_config.php` to point to the new location.

### Adding Analyzers, Tokenizers and Token Filters

When a document is indexed, its individual fields are subject to the analyzing and tokenizing filters that can transform and normalize the data in the fields. For example — removing blank spaces, removing html code, stemming, removing a particular character and replacing it with another 
(see [Solr Wiki](http://wiki.apache.org/solr/AnalyzersTokenizersTokenFilters)).

Example: Replace synonyms on indexing (e.g. "i-pad" with "iPad")

	<?php
	class MyIndex extends SolrIndex {
		function init() {
			$this->addClass('Page');
			$this->addField('Content');
			$this->addAnalyzer('Content', 'filter', array('class' => 'solr.SynonymFilterFactory'));
		}
	}

	// Generates the following XML schema definition:
	// <field name="Page_Content" ...>
	//   <filter class="solr.SynonymFilterFactory" synonyms="syn.txt" ignoreCase="true" expand="false"/>
	// </field>

### Highlighting

Solr can highlight the searched terms in context of the matched content,
to help users determine the relevancy of results (e.g. in which part of a sentence
the term is used). In order to use this feature, the full content of the
field to be highlighted needs to be stored in the index,
by declaring it through `addStoredField()`.

	<?php
	class MyIndex extends SolrIndex {
		function init() {
			$this->addClass('Page');
			$this->addAllFulltextFields();
			$this->addStoredField('Content');
		}
	}

To search with highlighting enabled, you need to pass in a custom query parameter.
There's a lot more parameters to tweak results on the [Solr Wiki](http://wiki.apache.org/solr/HighlightingParameters).
	
	$index = new MyIndex();
	$query = new SearchQuery();
	$query->search('My Term');
	$results = $index->search($query, -1, -1, array('hl' => 'true'));

Each result will automatically contain an "Excerpt" property
which you can use in your own results template.
The searched term is highlighted with an `<em>` tag by default.

Note: It is recommended to strip out all HTML tags and convert entities on the indexed content,
to avoid matching HTML attributes, and cluttering highlighted content with unparsed HTML.

### Spell Checking ("Did you mean...")

Solr has various spell checking strategies (see the ["SpellCheckComponent" docs](http://wiki.apache.org/solr/SpellCheckComponent)), all of which are configured through `solrconfig.xml`.
In the default config which is copied into your index,
spell checking data is collected from all fulltext fields
(everything you added through `SolrIndex->addFulltextField()`).
The values of these fields are collected in a special `_text` field.

	$index = new MyIndex();
	$query = new SearchQuery();
	$query->search('My Term');
	$params = array('spellcheck' => 'true', 'spellcheck.collate' => 'true');
	$results = $index->search($query, -1, -1, $params);
	$results->spellcheck

The built-in `_text` data is better than nothing, but also has some problems:
Its heavily processed, for example by stemming filters which butcher words.
So misspelling "Govnernance" will suggest "govern" rather than "Governance".
This can be fixed by aggregating spell checking data in a separate

	<?php
	class MyIndex extends SolrIndex {

		function init() {
			// ...
			$this->addCopyField('SiteTree_Title', 'spellcheckData');
			$this->addCopyField('DMSDocument_Title', 'spellcheckData');
			$this->addCopyField('SiteTree_Content', 'spellcheckData');
			$this->addCopyField('DMSDocument_Content', 'spellcheckData');
		}

		// ...

		function getFieldDefinitions() {
			$xml = parent::getFieldDefinitions();
			
			$xml .= "\n\n\t\t<!-- Additional custom fields for spell checking -->";
			$xml .= "\n\t\t<field name='spellcheckData' type='textSpell' indexed='true' stored='false' multiValued='true' />";

			return $xml;
		}

	}

Now you need to tell solr to use our new field for gathering spelling data.
In order to customize the spell checking configuration,
create your own `solrconfig.xml` (see "File-based configuration").
In there, change the following directive:

	<!-- ... -->
	<searchComponent name="spellcheck" class="solr.SpellCheckComponent">
		<!-- ... -->
		<str name="field">spellcheckData</str>
	</searchComponent

Don't forget to copy the new configuration via a call to the `Solr_Configure`
task, and reindex your data before using the spell checker.	

### Custom Types

Solr supports custom field type definitions which are written to its XML schema.
Many standard ones are already included in the default schema.
As the XML file is generated dynamically, we can add our own types
by overloading the template responsible for it: `types.ss`.

In the following example, we read out type definitions
from a new file `mysite/solr/templates/types.ss` instead:

	<?php
	class MyIndex extends SolrIndex {
		function getTemplatesPath() {
			return Director::baseFolder() . '/mysite/solr/templates/';
		}
	}

### File-based configuration (solrconfig.xml etc)

Many aspects of Solr are configured outside of the `schema.xml` file
which SilverStripe generates based on the index PHP file.
For example, stopwords are placed in their own `stopwords.txt` file,
and spell checks are configured in `solrconfig.xml`.

By default, these files are copied from the `fulltextsearch/conf/extras/`
directory over to the new index location. In order to use your own files,
copy these files into a location of your choosing (for example `mysite/data/solr/`),
and tell Solr to use this folder with the `extraspath` configuration setting.
	
	// mysite/_config.php
	Solr::configure_server(array(
		// ...
		'extraspath' => Director::baseFolder() . '/mysite/data/solr/',
	));

Please run the `Solr_configure` task for the changes to take effect.

Note: You can also define those on an index-by-index basis by
implementing `SolrIndex->getExtrasPath()`.

## Debugging

### Using the web admin interface

You can visit `http://localhost:8983/solr`, which will show you a list
to the admin interfaces of all available indices.
There you can search the contents of the index via the native SOLR web interface.

It is possible to manually replicate the data automatically sent 
to Solr when saving/publishing in SilverStripe, 
which is useful when debugging front-end queries, 
see `thirdparty/solr/server/silverstripe-solr-test.xml`.

	java -Durl=http://localhost:8983/solr/MyIndex/update/ -Dtype=text/xml -jar post.jar silverstripe-solr-test.xml

## FAQ

### How do I use date ranges where dates might not be defined?

The Solr index updater only includes dates with values,
so the field might not exist in all your index entries.
A simple bounded range query (`<field>:[* TO <date>]`) will fail in this case.
In order to query the field, reverse the search conditions and exclude the ranges you don't want:

	// Wrong: Filter will ignore all empty field values
	$myQuery->filter(<field>, new SearchQuery_Range('*', <date>));
	// Better: Exclude the opposite range
	$myQuery->exclude(<field>, new SearchQuery_Range(<date>, '*'));
