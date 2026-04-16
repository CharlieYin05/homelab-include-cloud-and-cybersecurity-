# How to delete OCI VCN

## 步骤
  1. 尝试Delate All
  2. 如果失败寻找引用，遵循一下关系一个个删：
  Subnet
    ↓
Route Table
    ↓
0.0.0.0/0 → Internet Gateway（把这里的Route Rules删了）
    ↓
Internet Gateway
    ↓
   VCN

## 坑：VNIC
❗很多资源会偷偷创建 VNIC

比如：
instance（明显）
load balancer（要注意）
database
file storage
