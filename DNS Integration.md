On godaddy we need to set two records that is A and CNAME. 
On vercel we need to select production when putting the custom domain and also then after we get A and CNAME records which we use to record the entries at godaddy.
eg.

a	@	216.198.79.1	600 seconds		
cname	www	971ce2a22bc241bc.vercel-dns-017.com.	1 Hour

Subdomain's process 

On godaddy we need to just add CNAME record.
On vercel we will get CNAME record.
eg1.
cname	vayu	82825417a0e46a03.vercel-dns-017.com.	600 seconds

eg2.
cname	letsgo	iamomjoshi.github.io.	600 seconds
