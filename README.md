using UnityEngine;
using System.Collections;

[RequireComponent(typeof(Rigidbody2D))]
public class PlayerController : MonoBehaviour
{
    Rigidbody2D rb;

    [Header("Movement")]
    public float moveSpeed = 7f;

    [Header("Jump")]
    public float jumpForce = 12f;
    [Tooltip("Second jump is scaled by this multiplier")]
    public float doubleJumpMultiplier = 0.75f;

    [Header("Wall")]
    public float wallSlideSpeed = -2f;
    public float wallJumpHorizontalForce = 14f;
    public float wallJumpVerticalVelocity = 10f;
    public float wallJumpLockTime = 0.18f;

    [Header("Dash")]
    public float dashForce = 18f;
    public float dashDuration = 0.18f;
    public float dashCooldown = 2.5f;

    // --- State ---
    bool isGrounded;
    bool canDoubleJump;
    bool isTouchingWall;
    bool canWallJump;
    bool hasUsedWallJump;
    bool isDashing;

    float moveInput;
    float wallJumpTimer = 0f;
    float dashCooldownTimer = 0f;

    Vector2 mostHorizontalWallNormal = Vector2.zero;

    void Awake()
    {
        rb = GetComponent<Rigidbody2D>();
        rb.collisionDetectionMode = CollisionDetectionMode2D.Continuous;
        rb.freezeRotation = true;
    }

    void Update()
    {
        moveInput = Input.GetAxisRaw("Horizontal");

        // --- Jump input ---
        if (Input.GetKeyDown(KeyCode.Space))
        {
            // Wall jump (priority)
            if (isTouchingWall && canWallJump && !isGrounded)
            {
                DoWallJump();
                return;
            }

            // Ground jump
            if (isGrounded)
            {
                rb.linearVelocity = new Vector2(rb.linearVelocity.x, jumpForce);
                canDoubleJump = true;
                return;
            }

            // Double jump
            if (canDoubleJump && !isTouchingWall)
            {
                rb.linearVelocity = new Vector2(rb.linearVelocity.x, jumpForce * doubleJumpMultiplier);
                canDoubleJump = false;
            }
        }

        // --- Dash input ---
        if (Input.GetKeyDown(KeyCode.Q) && !isDashing && dashCooldownTimer <= 0f)
        {
            StartCoroutine(PerformDash());
        }

        // Timers
        if (wallJumpTimer > 0f) wallJumpTimer -= Time.deltaTime;
        if (dashCooldownTimer > 0f) dashCooldownTimer -= Time.deltaTime;
    }

    void FixedUpdate()
    {
        // Skip horizontal control while dashing or wall jump locked
        if (!isDashing && wallJumpTimer <= 0f)
        {
            rb.linearVelocity = new Vector2(moveInput * moveSpeed, rb.linearVelocity.y);
        }

        // Wall slide
        if (isTouchingWall && !isGrounded && rb.linearVelocity.y < 0f)
        {
            rb.linearVelocity = new Vector2(rb.linearVelocity.x, Mathf.Max(rb.linearVelocity.y, wallSlideSpeed));
        }
    }

    // --- Wall Jump ---
    void DoWallJump()
    {
        wallJumpTimer = wallJumpLockTime;
        rb.linearVelocity = Vector2.zero;

        float pushDir = Mathf.Sign(mostHorizontalWallNormal.x);
        if (pushDir == 0f)
            pushDir = (moveInput != 0f) ? Mathf.Sign(moveInput) : 1f;

        rb.linearVelocity = new Vector2(pushDir * wallJumpHorizontalForce, wallJumpVerticalVelocity);

        hasUsedWallJump = true;
        canWallJump = false;
        canDoubleJump = false;
        isGrounded = false;
    }

    // --- Dash ---
    IEnumerator PerformDash()
    {
        isDashing = true;
        float originalGravity = rb.gravityScale;
        rb.gravityScale = 0f;

        float dir = (moveInput != 0f) ? Mathf.Sign(moveInput) : 1f;
        rb.linearVelocity = Vector2.zero;
        rb.AddForce(new Vector2(dir * dashForce, 0f), ForceMode2D.Impulse);

        yield return new WaitForSeconds(dashDuration);

        rb.gravityScale = originalGravity;
        isDashing = false;
        dashCooldownTimer = dashCooldown;
    }

    // --- Collision Handling (NORMAL-BASED) ---
    private void OnCollisionEnter2D(Collision2D collision)
    {
        EvaluateCollision(collision);
    }

    private void OnCollisionStay2D(Collision2D collision)
    {
        EvaluateCollision(collision);
    }

    private void OnCollisionExit2D(Collision2D collision)
    {
        // Reset when leaving collisions
        isGrounded = false;
        isTouchingWall = false;
        mostHorizontalWallNormal = Vector2.zero;
    }

    private void EvaluateCollision(Collision2D collision)
    {
        foreach (ContactPoint2D contact in collision.contacts)
        {
            Vector2 n = contact.normal;

            // Ground detection (normal pointing up)
            if (n.y > 0.5f)
            {
                isGrounded = true;
                canDoubleJump = true;
                hasUsedWallJump = false; // reset wall jump on ground
                return;
            }

            // Wall detection (normal pointing sideways)
            if (Mathf.Abs(n.x) > 0.5f && n.y < 0.5f)
            {
                isTouchingWall = true;
                mostHorizontalWallNormal = n;

                if (!hasUsedWallJump) canWallJump = true;
                return;
            }
        }
    }
}

