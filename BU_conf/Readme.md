# Configuration to make Bitcoin Unlimited BCH nodes follow ABC relay rules
@im_uname

**Last updated: 20190326**

Any BUCash operator can take the bitcoin.conf, change any relevant parts in it and apply directly to `/.bitcoin` and make it conform to Bitcoin ABC relay rules.

Aligning relay rules is beneficial for the network: It strengthens zero-confirmation security by preventing reverse respends (see https://gist.github.com/imaginaryusername/edcd611313abb5390872b7dc4911d170). It may also result in blocks you mine conforming better to prevailing mempools on the network, and propagating faster. If you are a merchant, this can prevent complications in service if a low fee or nonstandard transaction hits your node and does not get confirmed for a long time.

This configuration is not rigorously tested; the author has only done some rudimentary trials by manual transactions. If you find any unexpected behavior, please report.

The author will not be responsible for any potential complications or losses that results from using this configuration. 
