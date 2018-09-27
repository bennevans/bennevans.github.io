---
layout: post
use_math: true
---

Work In Progress!

To evaluate the performance of the network, I created 4 datasets of sizes 10k, 100k, 1m, and 5m warped images and trained the network on them. I also created a separate dataset of 10k warped images for cross validation (I realized I used the same random seed for each dataset, so both the 10k datasets are the same. VERY WRONG NEED TO FIX; might be cause of issues).

While the train loss decreased, the validation loss seems to increase, indicating overfitting. The 10k training does the best because the validation set is the same as the train set.

![train_loss](/assets/datasets/train_loss.png)
![val_loss](/assets/datasets/val_loss.png)