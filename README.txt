Fragmentos do Vazio - Projeto de Jogo 2D

Este jogo foi criado com Unity como parte do portfólio de desenvolvimento de jogos.
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    [Header("Componentes")]
    private Rigidbody2D rb;
    private Animator animator;
    private SpriteRenderer spriteRenderer;
    private BoxCollider2D boxCollider;
    private AudioSource audioSource;

    [Header("Movimento")]
    [SerializeField] private float moveSpeed = 7f;
    [SerializeField] private float runSpeedMultiplier = 1.5f;
    private float horizontalInput;
    private bool isFacingRight = true;
    private bool isRunning = false;

    [Header("Pulo")]
    [SerializeField] private float jumpForce = 14f;
    [SerializeField] private LayerMask groundLayer;
    [SerializeField] private Transform groundCheck;
    [SerializeField] private float groundCheckRadius = 0.2f;
    private bool isGrounded;
    private bool canDoubleJump = false;
    [SerializeField] private AudioClip jumpSound;

    [Header("Dash")]
    [SerializeField] private float dashForce = 24f;
    [SerializeField] private float dashDuration = 0.2f;
    [SerializeField] private float dashCooldown = 1f;
    private bool isDashing = false;
    private bool canDash = true;
    [SerializeField] private AudioClip dashSound;
    [SerializeField] private ParticleSystem dashEffect;

    [Header("Ataque")]
    [SerializeField] private Transform attackPoint;
    [SerializeField] private float attackRange = 1f;
    [SerializeField] private LayerMask enemyLayer;
    [SerializeField] private float attackDamage = 1f;
    [SerializeField] private float attackCooldown = 0.5f;
    private bool canAttack = true;
    [SerializeField] private AudioClip attackSound;
    [SerializeField] private ParticleSystem attackEffect;

    [Header("Status")]
    [SerializeField] private int maxHealth = 3;
    private int currentHealth;
    private int fragmentsCollected = 0;
    private bool isDead = false;
    [SerializeField] private AudioClip hurtSound;
    [SerializeField] private AudioClip fragmentCollectSound;
    [SerializeField] private AudioClip deathSound;

    // Referência à UI para atualizar contagem de fragmentos e vida
    [SerializeField] private GameUI gameUI;

    void Start()
    {
        // Inicializa componentes
        rb = GetComponent<Rigidbody2D>();
        animator = GetComponent<Animator>();
        spriteRenderer = GetComponent<SpriteRenderer>();
        boxCollider = GetComponent<BoxCollider2D>();
        audioSource = GetComponent<AudioSource>();
        
        currentHealth = maxHealth;
        
        // Atualiza a UI inicial
        if (gameUI != null)
        {
            gameUI.UpdateHealthUI(currentHealth);
            gameUI.UpdateFragmentsUI(fragmentsCollected);
        }
    }

    void Update()
    {
        if (isDead || isDashing) return;
        
        // Movimento horizontal
        horizontalInput = Input.GetAxisRaw("Horizontal");
        
        // Verificação se está no chão
        isGrounded = Physics2D.OverlapCircle(groundCheck.position, groundCheckRadius, groundLayer);
        
        // Controle de corrida
        isRunning = Input.GetKey(KeyCode.LeftShift);
        
        // Flip do sprite
        if (horizontalInput > 0 && !isFacingRight)
            Flip();
        else if (horizontalInput < 0 && isFacingRight)
            Flip();
        
        // Pulo
        if (Input.GetKeyDown(KeyCode.Space))
        {
            if (isGrounded)
            {
                Jump();
                canDoubleJump = true;
            }
            else if (canDoubleJump)
            {
                DoubleJump();
                canDoubleJump = false;
            }
        }
        
        // Dash
        if (Input.GetKeyDown(KeyCode.LeftControl) && canDash)
        {
            StartCoroutine(Dash());
        }
        
        // Ataque
        if (Input.GetMouseButtonDown(0) && canAttack)
        {
            StartCoroutine(Attack());
        }
        
        // Atualiza animações
        UpdateAnimations();
    }
    
    void FixedUpdate()
    {
        if (isDead || isDashing) return;
        
        // Aplica movimento horizontal
        float speedMultiplier = isRunning ? runSpeedMultiplier : 1f;
        rb.velocity = new Vector2(horizontalInput * moveSpeed * speedMultiplier, rb.velocity.y);
    }
    
    void Flip()
    {
        isFacingRight = !isFacingRight;
        transform.localScale = new Vector3(-transform.localScale.x, transform.localScale.y, transform.localScale.z);
    }
    
    void Jump()
    {
        rb.velocity = new Vector2(rb.velocity.x, jumpForce);
        PlaySound(jumpSound);
    }
    
    void DoubleJump()
    {
        rb.velocity = new Vector2(rb.velocity.x, jumpForce * 0.8f);
        PlaySound(jumpSound);
        // Efeito visual para o pulo duplo
        if (dashEffect != null)
        {
            ParticleSystem effect = Instantiate(dashEffect, transform.position, Quaternion.identity);
            effect.Play();
            Destroy(effect.gameObject, effect.main.duration);
        }
    }
    
    IEnumerator Dash()
    {
        canDash = false;
        isDashing = true;
        
        // Salva a gravidade original e desativa temporariamente
        float originalGravity = rb.gravityScale;
        rb.gravityScale = 0;
        
        // Aplica força do dash na direção que o jogador está olhando
        float dashDirection = isFacingRight ? 1f : -1f;
        rb.velocity = new Vector2(dashDirection * dashForce, 0);
        
        // Efeito de dash
        if (dashEffect != null)
        {
            dashEffect.Play();
        }
        
        PlaySound(dashSound);
        
        yield return new WaitForSeconds(dashDuration);
        
        // Restaura a gravidade
        rb.gravityScale = originalGravity;
        isDashing = false;
        
        // Cooldown do dash
        yield return new WaitForSeconds(dashCooldown);
        canDash = true;
    }
    
    IEnumerator Attack()
    {
        canAttack = false;
        
        // Executa o ataque
        if (attackEffect != null)
        {
            attackEffect.Play();
        }
        
        PlaySound(attackSound);
        
        // Detecta inimigos no alcance do ataque
        Collider2D[] hitEnemies = Physics2D.OverlapCircleAll(attackPoint.position, attackRange, enemyLayer);
        foreach (Collider2D enemy in hitEnemies)
        {
            // Aplica dano ao inimigo
            enemy.GetComponent<EnemyController>()?.TakeDamage(attackDamage);
        }
        
        // Cooldown do ataque
        yield return new WaitForSeconds(attackCooldown);
        canAttack = true;
    }
    
    public void CollectFragment()
    {
        fragmentsCollected++;
        PlaySound(fragmentCollectSound);
        
        // Atualiza UI
        if (gameUI != null)
        {
            gameUI.UpdateFragmentsUI(fragmentsCollected);
        }
    }
    
    public void TakeDamage(int damage)
    {
        if (isDead) return;
        
        currentHealth -= damage;
        PlaySound(hurtSound);
        
        // Atualiza UI
        if (gameUI != null)
        {
            gameUI.UpdateHealthUI(currentHealth);
        }
        
        // Efeito de dano (piscar)
        StartCoroutine(DamageEffect());
        
        // Verifica se o jogador morreu
        if (currentHealth <= 0)
        {
            Die();
        }
    }
    
    void Die()
    {
        isDead = true;
        PlaySound(deathSound);
        
        // Desativa controles
        rb.velocity = Vector2.zero;
        boxCollider.enabled = false;
        
        // Inicia animação de morte
        animator.SetTrigger("Die");
        
        // Game over após um delay
        StartCoroutine(GameOver());
    }
    
    IEnumerator GameOver()
    {
        yield return new WaitForSeconds(2f);
        
        // Implementar lógica de game over aqui
        // GameManager.instance.GameOver();
    }
    
    IEnumerator DamageEffect()
    {
        // Faz o sprite piscar quando toma dano
        for (int i = 0; i < 3; i++)
        {
            spriteRenderer.color = new Color(1, 0.5f, 0.5f, 0.8f);
            yield return new WaitForSeconds(0.1f);
            spriteRenderer.color = Color.white;
            yield return new WaitForSeconds(0.1f);
        }
    }
    
    void UpdateAnimations()
    {
        // Atualize o Animator com os estados do jogador
        animator.SetFloat("Speed", Mathf.Abs(horizontalInput));
        animator.SetBool("IsGrounded", isGrounded);
        animator.SetBool("IsRunning", isRunning);
        animator.SetBool("IsDashing", isDashing);
    }
    
    void PlaySound(AudioClip clip)
    {
        if (clip != null && audioSource != null)
        {
            audioSource.clip = clip;
            audioSource.Play();
        }
    }
    
    // Desenha gizmos para visualizar áreas de colisão no editor
    void OnDrawGizmosSelected()
    {
        if (groundCheck != null)
        {
            Gizmos.color = Color.yellow;
            Gizmos.DrawWireSphere(groundCheck.position, groundCheckRadius);
        }
        
        if (attackPoint != null)
        {
            Gizmos.color = Color.red;
            Gizmos.DrawWireSphere(attackPoint.position, attackRange);
        }
    }
}
