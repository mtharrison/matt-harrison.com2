---
title: "Keeping on top of invoices with the Freshbooks API and Ruby"
date: 2014-03-24T00:00:00+01:00
---
Freshbooks is a great tool as a freelancer, for managing your billing and sending invoices. It is however lacking in a few places for me.

One thing I really want to see when I log in is how many days have elapsed since I sent each invoice out. Yes I could work this out from the dates, but I don't have the time quite often so things start slipping.

Luckily there's an API we can use, and I knocked up this Ruby script in 10 minutes to give me a nicely formatted view of all my active invoices and the days since I created them. There's no field in the XML output for *days since sent*, and I often prepare an invoice before sending it. However it will have to do until they change that.

I've used the gem `terminal-table` to produce an attractive CLI table output:

<pre><code class="language-markup">+------------+--------------+-------------+---------------------+
| Invoice no | Client       | Outstanding | Days since invoiced |
+------------+--------------+-------------+---------------------+
| 0000054    | ACME Company | £1000.50    | 3                   |
| 0000053    | ACME Company | £20.00      | 5                   |
| 0000052    | ACME Company | £811.00     | 11                  |
| 0000051    | ACME Company | £12.00      | 14                  |
| 0000050    | ACME Company | £36.00      | 14                  |
| 0000049    | ACME Company | £37.00      | 24                  |
| 0000045    | ACME Company | £70.00      | 26                  |
| 0000041    | ACME Company | £700.00     | 35                  |
+------------+--------------+-------------+---------------------+
| Total Owed |              | £2686.50    |                     |
+------------+--------------+-------------+---------------------+
</code></pre>

Here's the script, I hope it's of some use to somebody else (Note: Please remove the additional ';'s within the XML.)

<pre><code class="language-ruby language-markup">#!/bin/env ruby
# encoding: utf-8

require 'xmlsimple'
require 'terminal-table'
require 'date'

API_TOKEN = "" #Put your freshbooks api token here
API_URL = "" #Put your freshbooks api url
CURRENCY_SYMBOL = "£"

xml = '&lt?xml version=&quot;1.0&quot; encoding=&quot;utf-8&quot;?&gt;
&lt;request method=&quot;invoice.list&quot;&gt;
  &lt;folder&gt;active&lt;/folder&gt;
&lt;/request&gt;'
response = `curl -s -u #{API_TOKEN}:X #{API_URL} -d '#{xml}'`
data = (XmlSimple.xml_in response)['invoices'][0]['invoice']
owed = 0

table = Terminal::Table.new do |t|
  t.add_row [
    'Invoice no',
    'Client',
    'Outstanding',
    'Days since invoiced'
  ]

  t.add_separator

  data.each do |i|
    t.add_row [
      i['number'][0],
      i['organization'][0],
      CURRENCY_SYMBOL + i['amount_outstanding'][0],
      (Date.today - Date.parse(i['date'][0])).to_i
    ]
    owed += i['amount_outstanding'][0].to_f
  end

  t.add_separator
  t.add_row [
    'Total Owed',
    '',
    CURRENCY_SYMBOL + owed.to_s,
    ''
  ]
end

puts table
</code></pre>

I've also added this to my path so I can call it anywhere with just the command `invoices`:

<pre><code class="language-markup">sudo ln -s /path/to/ruby/script.rb /usr/local/bin/invoices && chmod u+x /usr/local/bin/invoices</code></pre>
