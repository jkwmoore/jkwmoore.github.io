---
layout: post
title: "Puppet PDK testing: Show rendered templates while testing locally"
category: Puppet
---

### What was the problem?

I want to see what my templates are rendering while doing local testing. This is pretty helpful especially if you've got some hiera data being used
that you want to ensure is being rendered correctly.

### How can we show the rendered templates during PDK testing?

Turns out this is pretty easy (so long as you like cheap and kind of nasty) and needs only the most basic changes to your class's spec file.

e.g. to show a specific template's content via the CLI output, add the following in your class spec file, ``spec/classes/myclass_spec.rb``:

```ruby
require 'spec_helper'

describe 'mymodule::myclass' do
  on_supported_os.each do |os, os_facts|
    context "on #{os}" do
      let(:facts) { os_facts }

      it { is_expected.to compile }

      # To visualise the output of the /the/path/to/your/defined/file/resource.txt template during a PDK run, 
      # set TEMPLATE_DEBUG=true in your shell environment before running ``pdk test unit``.
      # i.e. ``TEMPLATE_DEBUG=true pdk test unit``

      it 'echoes rendered template content with OS info and markers (if TEMPLATE_DEBUG is set true)' do
        if ENV['TEMPLATE_DEBUG'] == 'true'
          content = catalogue.resource('file', '/the/path/to/your/defined/file/resource.txt').send(:parameters)[:content]
          puts "################################ Generated /the/path/to/your/defined/file/resource.txt content on: #{os}"
          puts "#{content}"
          puts "################################"
        end
      end

    end
  end
end
```


This might be pretty noisy if you're PDK testing against a lot of OSes with multiple templates though ``¯\_(ツ)_/¯``, so I
figure why not be a bit more clever about it and simply render out all the template content to a subdirectory for inspection?


```ruby
require 'spec_helper'

describe 'mymodule::myclass' do
  on_supported_os.each do |os, os_facts|
    context "on #{os}" do
      let(:facts) { os_facts }

      it { is_expected.to compile }

      # To render out all templated content during a PDK run to the ``rendered_templates`` subdirectory,
      # set TEMPLATE_DEBUG=true in your shell environment before running ``pdk test unit``.
      # i.e. ``TEMPLATE_DEBUG=true pdk test unit``

      it 'writes generated file content to rendered_templates directory per OS version' do
        # Check if TEMPLATE_DEBUG is set to 'true'
        if ENV['TEMPLATE_DEBUG'] == 'true'
          os_name = facts[:os]['name'].downcase
          os_version = facts[:os]['release']['full']
          # Write out the content per OS version
          base_path = File.join('rendered_templates', "#{os_name}-#{os_version}")
      
          catalogue.resources.each do |resource|
            # Only process 'File' resources with a valid path.
            # This way we don't try to dump directories.
            next unless resource.type == 'File'

            path = resource[:path] || resource.title
            content = resource[:content]
            
            if path && content
              output_path = File.join(base_path, path)
      
              # Create the directory if it doesn't exist
              FileUtils.mkdir_p(File.dirname(output_path))

              # Write the content to the file
              File.write(output_path, content)
              puts "✅ Wrote content to: #{output_path}"
            elsif path
              # Note - if the content is sourced from a file, the resource catalog does not have a content
              # element for us to render out.
              puts "⚠️ No content for: #{path} (likely sourced)"
            end
          end
        end
      end

    end
  end
end
```

I've found this to be a useful method during PDK testing for confirming correct template rendering while working with hiera data (structures). 

This has also been particularly useful for visualising the created directories, files and content if the classes you are creating are calling 
other classes, as their outputs will also be rendered!

This is all very helpful for determining and implementing new spec tests for your classes, e.g. for adding tests to automatically validate
template / file contents generated by the subordinate classes.

GLHF!

JKWM.
