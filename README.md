# ActiveAdminImport 
The most fastest and efficient CSV import for Active Admin (based on activerecord-import gem) 
with support of validations and bulk inserts 



[![Build Status](http://img.shields.io/travis/Fivell/active_admin_import.svg)](https://travis-ci.org/Fivell/active_admin_import)
[![Dependency Status](http://img.shields.io/gemnasium/Fivell/active_admin_import.svg)](https://gemnasium.com/Fivell/active_admin_import)
[![Coverage Status](https://coveralls.io/repos/Fivell/active_admin_import/badge.svg?branch=3.0.0)](https://coveralls.io/r/Fivell/active_admin_import?branch=3.0.0)

 branch for AA 1.0.0 and Rails >= 4.1


#Installation

Add this line to your application's Gemfile:

```ruby
gem "active_admin_import" , '~> 3.0.0'
```
	
And then execute:

    $ bundle



#Why yet another import for ActiveAdmin ? Now with activerecord-import ....

 <p>Because plain-vanilla, out-of-the-box ActiveRecord doesn’t provide support for inserting large amounts of data efficiently</p>

Features of activerecord-import

<ol>
  <li>activerecord-import can perform validations (fast)</li>
  <li>activerecord-import can perform on duplicate key updates (requires mysql)</li>
</ol>

    
    


# active_admin_import features
<ol>
  <li>Encoding handling</li>
  <li>Preview before importing (Example 2)</li>
  <li> CSV options</li>
  <li> Ability to prepend CSV headers automatically</li>
  <li>Bulk import (activerecord-import)</li>
  <li>Callbacks</li>
  <li>Zip files</li>
  <li>more...</li>
</ol>

   


#### Options

   <table>
<tr><td>name</td><td>description</td></tr>
<tr><td>:back</td><td>resource action to redirect after processing</td></tr>
<tr><td>:csv_options</td><td>hash with column separator, row separator, etc </td></tr>
<tr><td>:validate</td><td>true|false, means perform validations or not</td></tr>
<tr><td>:batch_size</td><td>integer value of max  record count inserted by 1 query/transaction</td></tr>
<tr><td>:before_import</td><td>proc for before import action, hook called with  importer object</td></tr>
<tr><td>:after_import</td><td>proc for after import action, hook called with  importer object</td></tr>
<tr><td>:before_batch_import</td><td>proc for before each batch action, called with  importer object</td></tr>
<tr><td>:after_batch_import</td><td>proc for after each batch action, called with  importer object</td></tr>
<tr><td>:on_duplicate_key_update</td><td>an Array or Hash, tells activerecord-import to use MySQL's ON DUPLICATE KEY UPDATE ability.</td></tr>
<tr><td>:timestamps</td><td>true|false, tells activerecord-import to not add timestamps (if false) even if record timestamps is disabled in ActiveRecord::Base</td></tr>
<tr><td>:ignore</td><td>true|false, tells activerecord-import toto use MySQL's INSERT IGNORE ability</td></tr>
<tr><td>:template</td><td>custom template rendering</td></tr>
<tr><td>:template_object</td><td>object passing to view</td></tr>
<tr><td>:locals</td><td>more variables for template</td></tr>
<tr><td>:resource_class</td><td>resource class name</td></tr>
<tr><td>:resource_label</td><td>resource label value</td></tr>
<tr><td>:plural_resource_label</td><td>pluralized resource label value (default config.plural_resource_label)</td></tr>
<tr><td>:headers_rewrites</td><td>hash with key (csv header) - value (db column name) rows mapping</td></tr>
</table>



#### Default options values

```ruby    
    back: {action: :import},
    csv_options: {},
    template: "admin/import",
    fetch_extra_options_from_params: [],
    resource_class: config.resource_class,
    resource_label:  config.resource_label,
    plural_resource_label: config.plural_resource_label,
```    

#### Example1 

```ruby  
    ActiveAdmin.register Post  do
       active_admin_import  validate: false,
                            csv_options: {col_sep: ";" },
                            before_import: proc{ Post.delete_all},
                            batch_size: 1000
    
    
    end
```


#### Example2 Importing to mediate table with insert select operation after import completion

<p> This config allows to replace data in 1 sql query with callback </p>

```ruby
    ActiveAdmin.register Post  do
        active_admin_import validate: false,
            csv_options: {col_sep: ";" },
            resource_class: ImportedPost ,  # we import data into another resource
            before_import: proc{ ImportedPost.delete_all },
            after_import: proc{
                Post.transaction do
                    Post.delete_all
                    Post.connection.execute("INSERT INTO posts (SELECT * FROM imported_posts)")
                end
            },
            back: proc { config.namespace.resource_for(Post).route_collection_path } # redirect to post index
    end
```


#### Example3 Importing file without headers, but we always know file format, so we can predefine it

```ruby
    ActiveAdmin.register Post  do
        active_admin_import validate: true,
            template_object: ActiveAdminImport::Model.new(
                hint: "file will be imported with such header format: 'body','title','author'",
                csv_headers: ["body","title","author"]
            )
    end
```
 
#### Example4 Importing without forcing to UTF-8 and disallow archives


```ruby
    ActiveAdmin.register Post  do
        active_admin_import validate: true,
            template_object: ActiveAdminImport::Model.new(
                hint: "file will be encoded to ISO-8859-1",
                force_encoding: "ISO-8859-1",
                allow_archive: false
            )
    end
```


#### Example5 Callbacks for each bulk insert iteration


```ruby
    ActiveAdmin.register Post  do
        active_admin_import validate: true,
        before_batch_import: proc { |import|
           import.file #current file used
           import.resource #ActiveRecord class to import to
           import.options # options
           import.result # result before bulk iteration
           import.headers # CSV headers
           import.csv_lines #lines to import
           import.model #template_object instance
        },
        after_batch_import: proc{ |import|
           #the same
        }
    end
```    
    
#### Example6 dynamic CSV options, template overriding

 -  put overrided template to ```app/views/import.html.erb```

```erb

    <p>
       <%= raw(@active_admin_import_model.hint) %> 
    </p>
    <%= semantic_form_for @active_admin_import_model, url: {action: :do_import}, html: {multipart: true} do |f| %>
        <%= f.inputs do %>
            <%= f.input :file, as: :file %>
        <% end %>
        <%= f.inputs "CSV options", for: [:csv_options, OpenStruct.new(@active_admin_import_model.csv_options)] do |csv| %>
            <% csv.with_options input_html: {style: 'width:40px;'} do |opts| %>
                <%= opts.input :col_sep %>
                <%= opts.input :row_sep %>
                <%= opts.input :quote_char %>
            <% end %>
        <% end %>
    
        <%= f.actions do %>
            <%= f.action :submit, label: t("active_admin_import.import_btn"), button_html: {disable_with: t("active_admin_import.import_btn_disabled")} %>
        <% end %>
    <% end %>
    
```

 - call method with following parameters

```ruby
    ActiveAdmin.register Post  do
        active_admin_import validate: false,
                          template: 'import' ,
                          template_object: ActiveAdminImport::Model.new(
                              hint: "specify CSV options"
                              csv_options: {col_sep: ";", row_sep nil, :quote_char => nil}
                          )
    end                      
```

#Links
https://github.com/gregbell/active_admin

https://github.com/zdennis/activerecord-import






