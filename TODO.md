**TODO**
- [X] Test with circle
- [X] Command should be optional - should use the dockerfile by default
- [X] Figure out multiline in YAML configuration
- [ ] Parameterize subnet and security groups from TaskDefinition (should go in AWS Secrets)
- [ ] Add robustness
	- [ ] if target group exists, delete
	- [ ] if rules exists, delete

**Maybe**
- [ ] Create custom log groups automatically
- [ ] Load options from JSON files?
- [ ] Do find/replace on task definition instead of setting JSON? (So custom task definition does not get overridden?)
- [ ] Do we need grace period?
