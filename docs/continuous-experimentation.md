# Canary release
## What's changed
As shown in the picture, we updated out frontend UI, we changed the background color and modified out input window and button.
![UI Comparison](https://github.com/user-attachments/assets/401dfd05-c130-4e8c-917e-9d133663e907)

## Hypothesis
Since the new UI looks better than original plain UI, and thus shall satisfy user more.
User is likely to input more messages. We have `metrics: sms_requests_total`. So the hypothesis is that with the new UI,
there will be increase in `metrics: sms_requests_total`.

## Decision Process
We will deploy stable deployment and canary deployment. And compare the `metrics: sms_requests_total`. If we found that 
in canary deployment, the user consistently have more `metrics: sms_requests_total`.
We'll increase percentage our canary deployment gradually.

## Screenshot of our metrics in Grafana

![Screenshot of out metrics in Grafana](https://github.com/user-attachments/assets/a24f0dec-9075-432f-9379-757438586148)