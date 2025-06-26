# Cert-Manager Nil Pointer Dereference Bug Fix

## Issue Description
**Bug Report**: [OCPBUGS-44563](https://issues.redhat.com/browse/OCPBUGS-44563)

The cert-manager readiness controller was experiencing nil pointer dereference panics when logging certificate validity fields. The error appeared in logs as:

```
I1113 16:08:47.256372       1 readiness_controller.go:191] "updating status fields" logger="cert-manager.certificates-readiness" key="test-cert-manager/debug-13h" notAfter="<panic: runtime error: invalid memory address or nil pointer dereference>" notBefore="<panic: runtime error: invalid memory address or nil pointer dereference>" renewalTime="<panic: runtime error: invalid memory address or nil pointer dereference>"
```

## Root Cause
The issue occurred in the `ProcessItem` method of the readiness controller (`pkg/controller/certificates/readiness/readiness_controller.go` line 197-199), where the logging statement directly referenced potentially nil pointer fields:

```go
log.V(logf.DebugLevel).Info("updating status fields", "notAfter",
    crt.Status.NotAfter, "notBefore", crt.Status.NotBefore, "renewalTime",
    crt.Status.RenewalTime)
```

This happened when:
1. The certificate's secret doesn't exist
2. The secret exists but doesn't contain valid certificate data
3. Certificate decoding fails

In these cases, the status fields are explicitly set to `nil`:
```go
crt.Status.NotAfter = nil
crt.Status.NotBefore = nil
crt.Status.RenewalTime = nil
```

## Solution
Modified the logging statement to safely handle nil pointers by creating string representations before logging:

```go
// Create safe string representations for logging to avoid nil pointer dereferences
var notAfterStr, notBeforeStr, renewalTimeStr string
if crt.Status.NotAfter != nil {
    notAfterStr = crt.Status.NotAfter.String()
} else {
    notAfterStr = "<nil>"
}
if crt.Status.NotBefore != nil {
    notBeforeStr = crt.Status.NotBefore.String()
} else {
    notBeforeStr = "<nil>"
}
if crt.Status.RenewalTime != nil {
    renewalTimeStr = crt.Status.RenewalTime.String()
} else {
    renewalTimeStr = "<nil>"
}

log.V(logf.DebugLevel).Info("updating status fields", "notAfter",
    notAfterStr, "notBefore", notBeforeStr, "renewalTime",
    renewalTimeStr)
```

## Files Modified
- `pkg/controller/certificates/readiness/readiness_controller.go` - Fixed the nil pointer dereference
- `pkg/controller/certificates/readiness/readiness_controller_test.go` - Added test case for nil status fields

## Testing
- Added a specific test case: "should safely log nil status fields without panicking"
- Existing tests already cover scenarios with nil status fields
- The fix maintains backward compatibility and doesn't change the controller's behavior

## Impact
- **Bug Severity**: Low - cosmetic issue in logs, doesn't affect cert-manager functionality
- **User Experience**: Eliminates ugly panic messages in logs
- **Compatibility**: No breaking changes, safe for all versions

## Verification
The fix can be verified by:
1. Creating a ClusterIssuer with ACME HTTP01 solver
2. Creating a Certificate that fails to get a valid certificate
3. Checking cert-manager logs - should show `<nil>` instead of panic messages 