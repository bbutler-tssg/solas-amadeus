
The next steps are

1.  get some of the latest CorMel data - I need to make some changes to my
code to load the extra fields that have been added to the Avro schema, so
it would be good to get that done soon;

2.  find how to label individual transaction trees according to whether
they have succeeded or failed. Joel has run some queries on the Transaction
status using the ElasticSearch web front end and provided some example data,
in JSON format to generate understanding;

3.  We need to be able to generate both CorMel and ElasticSearch data
that are cross-referenceable by `DcxId` and possibly other fields. In practice,
since the log data that ElasticSearch queries is never more than a few days old,
we need the CorMel extracts to restart.

4.  try to develop a model to predict whether a transaction tree has failed
or not, based on features derived from such transaction trees.

