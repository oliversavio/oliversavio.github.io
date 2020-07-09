---
layout: page
title: Buy Me a Beer
subtitle: Support me if you find my content helpful
---

#### Help me keep this blog up and running with a small contribution, INR 300 / USD 5 :)




<br />
<div>
<p>If you're in India you could use this <a href="https://www.instamojo.com/@oliver_savio/l9ce300019ebe43c492a2deb95dfa5fc7/">link</a></p>
</div>


<div>
<div id="paypal-button-container"></div>
<script src="https://www.paypal.com/sdk/js?client-id=sb&currency=USD" data-sdk-integration-source="button-factory"></script>
<script>
  paypal.Buttons({
      style: {
          shape: 'pill',
          color: 'white',
          layout: 'vertical',
          label: 'paypal',
          
      },
      createOrder: function(data, actions) {
          return actions.order.create({
              purchase_units: [{
                  amount: {
                      value: '5'
                  }
              }]
          });
      },
      onApprove: function(data, actions) {
          return actions.order.capture().then(function(details) {
              alert('Transaction completed by ' + details.payer.name.given_name + '!');
          });
      }
  }).render('#paypal-button-container');
</script>
</div>




