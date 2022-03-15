---
layout: post
title: "Encoding Lazy-Loaded Values in Chef for HTTP Authentication"
tags: chef lazy vault
date: '2022-03-15'
---

I spent the better part of 3 days banging my head against my desk trying to sort out HTTP authentication for [Chef](https://www.chef.io/)'s [`remote_file`](https://docs.chef.io/resources/remote_file/) resource. The method itself is straight-forward: you have to Base64 encode the username and password and specify that in a hash for the `headers` parameter for `remote_file`. However, we had a less-than-straight-forward situation in that our password was being read from our Vault instance into a `node.run_state['password']` variable at execution time, and so had to be [`lazy`](https://docs.chef.io/resource_common/#lazy-evaluation) loaded. With out `lazy` loading, the value returned by `node.run_state['password']` would always be empty. 

I'm not a Chef wizard by any means, so this was a frustrating few days. Thankfully I was able to pair up with a teammate and we banged our heads against our desks together and through a _lot_ of trial and error, we got it working.

It appeared simple enough to use the `lazy` keyword for simple string parameters like `source`, but the `headers` parameter took a hash value and that had us a little stumped. All of the classic StackOverflow examples just specified a username and password without dealing with `lazy` loading. I'm not going to walk you through every step of the trial-and-error, so here is the final version we had, `lazy` loading the entire hash:

```ruby
remote_file "/tmp/foo-v#{node['foo']['version']}.tar.gz" do
  source "https://internal-site.com/path/to/foo-v#{node['foo']['version']}.tar.gz"
  headers lazy {{ 'Authorization' => "Basic #{Base64.encode64("#{node['foo']['auth']['username']}:#{node.run_state['password']}").strip}" }}
  checksum node['foo']['checksum']
  mode '0755'
  action :create
end
```

One alternative that is possible with `remote_file` is to specify the username and password as part of the `source` URL and `lazy` load that, for example:

```ruby
  source lazy "https://#{node['foo']['auth']['username']}:#{node.run_state['password']}@internal-site.com/path/to/foo-v#{node['foo']['version']}.tar.gz"
```

We decided to use the `headers` instead since that seemed to be a little cleaner and the intended use.

I was so relieved to have found the working solution that I wanted to get it published, as this had been driving me nuts.
