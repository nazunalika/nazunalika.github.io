---
title: "IPA: Using it for external DNS (the awkward way)"
date: 2026-05-10T10:00:00-07:00
---

I've been in the process of redoing my IPA domain at home. I decided that I
wanted a new domain while not changing ID's, but also allowing me to control
some DNS domains that are external in the process. 

Traditionally, I have managed some domains with IPA while guaranteeing that some
are available externally and that if my IPA servers were down during
maintenance, bind on my AlmaLinux router would still serve DNS for my network.
This worked for the most part, actually, though I only had that one name server
that was known/usable. (You can imagine what happens when the one and only name
server for a domain stops responding to requests.)

This is where I wanted to do some clean up. Firstly, I wanted to combine my
entire /48 IPv6 subnet into a single rDNS zone. Secondly, I want to manage my
external domains that I know I won't use internally on my LAN. Thirdly, I wanted
to make sure that for the external domains I had more than one name server. For
this, I decided on Hurricane Electric's FreeDNS service to help.

### This doesn't seem like it should work

But it does. You can have non-IPA servers have slave zones that are managed that
way. All you have to do is add the non-IPA servers as NS records and then ensure
that AXFR's (zone transfers) are allowed to whatever IP's.

Example:

```
# Assumes an A record already exists, otherwise it will error out
# dns.example.com has an A record because it is an IPA client anyway.
ipa01$ ipa dnsrecord-add example.com '@' --ns-rec="dns.example.com."

# Make sure that this NS can AXFR for this zone
ipa01$ ipa dnszone-mod --allow-transfer="10.100.0.1" example.com
```

And then on the non-IPA server, assuming it's bind, a simple slave configuration
is fine.

```
        zone "example.com" {
                type slave;
                file "/var/named/network/internal/example.com.ipa.in";
                masterfile-format text;
                masters { ipa_servers_cw; };
        };
```

Once `named` has been restarted, the zone will transfer cleanly. Any changes
will come almost immediately when changed in IPA. This ensures that whether my
router box or my IPA servers are down, some DNS server should be there to answer
the query.

### What about for external-only domains?

This is where I had to do some thinking and reading through the man pages
(something I haven't done for bind in a minute). I wanted to make sure the
following happened:

* If a change happens on the IPA DNS servers, the update is seen internally
  immediately

* The change should *also* be immediately available externally and external
  name servers should get the update if they're listed in the NS records.

I originally thought in the external view, I could just point to the same exact
file that is being managed in the internal view. Turns out that this is not a
good idea and cause weird things to happen. My original compromise was to make
it "transfer" again. That doesn't address the second objective above.

Turns out, bind has an option called "in-zone" that helps facilitate this. And
since I use internal and external views, this should be straight forward. Let's
break it down.

```
# Assume example.net is external and nothing in the LAN will use this domain.
ipa01$ ipa dnsrecord-add example.net '@' --ns-rec="dns.example.com."
ipa01$ ipa dnsrecord-add example.net '@' --ns-rec="ns1.he.net."
ipa01$ ipa dnsrecord-add example.net '@' --ns-rec="ns2.he.net."
ipa01$ ipa dnsrecord-add example.net '@' --ns-rec="ns3.he.net."
ipa01$ ipa dnsrecord-add example.net '@' --ns-rec="ns4.he.net."
ipa01$ ipa dnsrecord-add example.net '@' --ns-rec="ns5.he.net."

# Make sure our non-IPA DNS server can transfer, of course.
ipa01$ ipa dnszone-mod --allow-transfer="10.100.0.1" example.net
```

On the bind

On the bind server, you need two slots, an ACL and master list (yes, this is
required even if the IP's are the same). My ACL and master list look like this:

```
acl he_trusted_only {
        216.218.133.2;
        2001:470:600::2;
};

masters he_trusted_notify {
        216.218.133.2;
        2001:470:600::2;
};
```

In the internal view, I have this:

```
        zone "example.net" {
                type slave;
                file "/var/named/network/internal/example.net.ipa.in";
                masterfile-format text;
                masters { ipa_servers_cw; };
                allow-transfer { he_trusted_only; };
                also-notify { he_trusted_notify; };
                notify yes;
        };
```

Essentially, even if the zone is in the "internal" view, you want your external
DNS servers to be able to AXFR. We also need to make sure that when there are
changes to `also-notify` those same DNS servers. And of course, `notify` is
`yes`.

In the external view, I have this:

```
        zone "example.net" {
                in-view "internal";
        };
```

And that's it. That's all that is needed. Now you just need to tell your
upstream DNS slave to AXFR.

### What does in-view actually do here?

The `in-view` directive allows multiple views to refer to the same in-memory
instance of a zone which includes the options/directives that are set.

Conceptually, the internal view owns the secondary zone and external view reuses
that same object, avoiding two independent transfers. 

The caveat: You **cannot** override or set other options when using it. Any
access controls and settings *must* be done in the originating view.

### Isn't this just Split-horizon DNS?

What I described above for `example.net` is not split-horizon DNS. This is
because the answers that are given, regardless of view, will be the same.

If I want `example.com` to be available externally, I would setup an entirely
separate zone that is not managed by IPA in any capacity. That would be
split-horizon at that point, as external queries will get a different answer for
"router.example.com" than if say a system in my internal view (LAN) asked about
it.

```
# The router's LAN address, as seen by the internal view
router$ dig @10.100.0.1 router.example.com A +short
10.100.0.1

# The router's external address, as seen by the internet
router$ dig @1.1.1.1 router.example.com A +short
X.X.X.X
```

### Final thoughts

Obviously this isn't the most elegant thing I've ever done, but it gets the job
done for what I want.
