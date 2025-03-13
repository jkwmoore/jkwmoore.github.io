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

e.g. in your class spec file, ``spec/classes/myclass_spec.rb``

```ruby
require 'spec_helper'

describe 'mymodule::myclass' do
  on_supported_os.each do |os, os_facts|
    context "on #{os}" do
      let(:facts) { os_facts }

      it { is_expected.to compile }

      # To visualise the output of the /the/path/to/your/defined/file/resource.txt template during a PDK run, 
      # set TEMPLATE_DEBUG=true in your shell environment before running ``pdk test unit``.
      # e.g. ``TEMPLATE_DEBUG=true pdk test unit``

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

This might be pretty noisy if you're PDK testing against a lot of OSes with multiple templates though ``¯\_(ツ)_/¯`` .

GLHF! JKWM.
