# S3 Object Tagging Support in Hadoop S3A Filesystem

## Overview

The Hadoop S3A filesystem connector now supports S3 object tagging, allowing users to automatically assign metadata tags to S3 objects during creation and soft deletion operations. This feature enables better data organization, cost allocation, access control, and lifecycle management for S3-stored data.

**JIRA Issue**: [HADOOP-19536](https://issues.apache.org/jira/browse/HADOOP-19536#s3-tags)

## Table of Contents

- [Motivation](#motivation)
- [S3 Object Tagging Capabilities](#s3-object-tagging-capabilities)
- [Use Cases](#use-cases)
- [Configuration](#configuration)
- [Usage Examples](#usage-examples)
- [Soft Delete Feature](#soft-delete-feature)
- [Best Practices](#best-practices)
- [Limitations](#limitations)

## Motivation

Amazon S3 supports tagging objects with key-value pairs, providing several critical benefits:

1. **Cost Allocation**: Track and allocate S3 storage costs across departments, projects, or cost centers
2. **Access Control**: Use tags in IAM policies to control object access permissions
3. **Lifecycle Management**: Trigger automated lifecycle policies for object transitions and expiration
4. **Data Classification**: Organize and classify data for compliance, security, and business requirements
5. **Analytics and Reporting**: Enable detailed analytics and reporting based on object metadata

Previously, the Hadoop S3A connector lacked native support for object tagging, requiring users to implement custom solutions or use separate tools to tag objects post-creation.

## S3 Object Tagging Capabilities

### Tag Specifications
- **Maximum Tags**: Up to 10 tags per object
- **Structure**: Key-value pairs
- **Key Length**: Up to 128 Unicode characters
- **Value Length**: Up to 256 Unicode characters
- **Case Sensitivity**: Keys and values are case-sensitive
- **Uniqueness**: Tag keys must be unique per object (no duplicate keys)

### Allowed Characters
Tag keys and values can contain:
- Letters (a-z, A-Z)
- Numbers (0-9)
- Spaces
- Special symbols: `. : + - = _ / @`

## Use Cases

### 1. Access Control with IAM Policies

Control object access based on tags:

```json
{
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource": "*",
    "Condition": {
        "StringEquals": {
            "s3:ExistingObjectTag/department": "finance"
        }
    }
}
```

### 2. Lifecycle Management

Trigger lifecycle rules based on tags:

```json
{
    "Rules": [
        {
            "Status": "Enabled",
            "Filter": {
                "Tag": {
                    "Key": "retention",
                    "Value": "temporary"
                }
            },
            "Expiration": {
                "Days": 30
            }
        }
    ]
}
```

### 3. Cost Allocation and Tracking

- Use tags for cost tracking in AWS Cost Explorer
- Allocate costs across different business units or projects
- Generate detailed billing reports by tag dimensions

### 4. Data Analytics and Filtering

- Use S3 Analytics to filter and analyze data by tags
- Create custom reports based on tagged object metadata
- Enable data governance and compliance reporting

## Configuration

### Object Creation Tags

#### Method 1: Comma-Separated List
```properties
fs.s3a.object.tags=department=finance,project=alpha,owner=data-team
```

#### Method 2: Individual Tag Properties
```properties
fs.s3a.object.tag.department=finance
fs.s3a.object.tag.project=alpha
fs.s3a.object.tag.owner=data-team
fs.s3a.object.tag.environment=production
```

### Soft Delete Tags
```properties
fs.s3a.soft.delete.enabled=true
fs.s3a.soft.delete.tag.key=archive
fs.s3a.soft.delete.tag.value=true
```

## Usage Examples

### Spark Applications

#### Using Comma-Separated Tags
```bash
spark-submit \
  --conf spark.hadoop.fs.s3a.object.tags=department=finance,project=alpha,environment=prod \
  --class MySparkApp \
  my-app.jar
```

#### Using Individual Tag Configurations
```bash
spark-submit \
  --conf spark.hadoop.fs.s3a.object.tag.department=finance \
  --conf spark.hadoop.fs.s3a.object.tag.project=alpha \
  --conf spark.hadoop.fs.s3a.object.tag.owner=data-team \
  --conf spark.hadoop.fs.s3a.object.tag.cost-center=engineering \
  --class MySparkApp \
  my-app.jar
```

### Hadoop Commands

#### File Upload with Tags
```bash
hadoop fs \
  -Dfs.s3a.object.tag.department=finance \
  -Dfs.s3a.object.tag.project=quarterly-report \
  -put local-file.txt s3a://my-bucket/reports/
```

#### Directory Operations with Tags
```bash
hadoop fs \
  -Dfs.s3a.object.tags=team=analytics,retention=long-term \
  -put /local/data/ s3a://my-bucket/analytics/
```

### MapReduce Jobs

```bash
hadoop jar my-job.jar \
  -Dfs.s3a.object.tag.job-type=etl \
  -Dfs.s3a.object.tag.priority=high \
  input s3a://my-bucket/output/
```

## Soft Delete Feature

The soft delete feature allows you to tag objects instead of permanently deleting them, enabling data retention policies and recovery options.

### Important Behavior Notes

- **Default Tags**: If no tag key and value are specified, default tags are used as defined in the configuration
- **Tag Replacement**: When soft delete is performed, **all existing tags on the object are removed** and replaced with only the soft delete tag specified by the user

### Current Implementation

```bash
# Using custom soft delete tags
hadoop fs \
  -Dfs.s3a.soft.delete.enabled=true \
  -Dfs.s3a.soft.delete.tag.key=archive \
  -Dfs.s3a.soft.delete.tag.value=true \
  -rm s3a://my-bucket/file-to-archive.txt

# Using default soft delete tags (if configured)
hadoop fs \
  -Dfs.s3a.soft.delete.enabled=true \
  -rm s3a://my-bucket/file-to-archive.txt
```

### Future Capabilities (Planned)

```bash
# Mark file as soft-deleted with default tags
hadoop fs -rm -softDelete s3a://bucket/path/to/file.txt

# Mark file as soft-deleted with custom tags
hadoop fs -rm -softDelete custom_status deleted s3a://bucket/path/to/file.txt

# List files (soft-deleted files won't appear)
hadoop fs -ls s3a://bucket/path/

# Permanently delete soft-deleted files (requires separate process)
# This would typically be done with S3 lifecycle rules or scheduled jobs
```

## Best Practices

### 1. Tag Naming Conventions
- Use consistent naming conventions across your organization
- Consider using prefixes for different tag categories (e.g., `cost:department`, `security:classification`)
- Use lowercase with hyphens for readability: `cost-center`, `data-classification`

### 2. Tag Management
- Document your tagging strategy and enforce it across teams
- Regularly audit and clean up unused or inconsistent tags
- Use automation to ensure consistent tagging

### 3. Cost Optimization
- Use tags to identify and optimize storage costs
- Implement lifecycle policies based on tags to automatically transition or delete objects
- Monitor tag-based cost allocation reports regularly

### 4. Security Considerations
- Use tags in IAM policies for fine-grained access control
- Avoid including sensitive information in tag values
- Regularly review tag-based access policies

## Limitations

### S3 Service Limits
- Maximum 10 tags per object
- Tag key length: 128 Unicode characters maximum
- Tag value length: 256 Unicode characters maximum
- No nested or hierarchical tag structures

### Performance Considerations
- Tagging adds minimal overhead to object creation operations
- Large numbers of tags may slightly impact performance
- Consider batching operations when possible

### Compatibility
- Feature requires S3A connector version with tagging support
- Some older Hadoop versions may not support all tagging features
- Verify compatibility with your specific Hadoop distribution

## Troubleshooting

### Common Issues

1. **Tag Validation Errors**
   - Ensure tag keys and values meet S3 character requirements
   - Check for duplicate tag keys
   - Verify tag count doesn't exceed 10 per object

2. **Permission Issues**
   - Ensure IAM permissions include `s3:PutObjectTagging` and `s3:GetObjectTagging`
   - Verify bucket policies allow tagging operations

3. **Configuration Problems**
   - Check property syntax and formatting
   - Ensure configuration properties are properly set in Hadoop configuration files

### Debug Commands

```bash
# Verify object tags using AWS CLI
aws s3api get-object-tagging --bucket my-bucket --key path/to/file.txt

# List objects with specific tags
aws s3api list-objects-v2 --bucket my-bucket --query "Contents[?contains(TagSet[?Key=='department'].Value, 'finance')]"
```

## Contributing

To contribute to this feature or report issues:

1. Check the [JIRA issue](https://issues.apache.org/jira/browse/HADOOP-19536) for current status
2. Follow Hadoop contribution guidelines
3. Submit patches through the Apache Hadoop review process
4. Include comprehensive tests for any new functionality

## References

- [Amazon S3 Object Tagging Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-tagging.html)
- [S3 Lifecycle Configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [IAM Policies with S3 Tags](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-tagging-managing.html)
- [Hadoop S3A Documentation](https://hadoop.apache.org/docs/current/hadoop-aws/tools/hadoop-aws/index.html)