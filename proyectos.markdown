---
layout: page
title: Proyectos
permalink: /proyectos/
---

<style>
  .project-container {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 2rem;
    margin-bottom: 2rem;
    padding-bottom: 2rem;
    border-bottom: 1px solid #333;
  }
  .project-image {
    flex: 1 1 200px;
  }
  .project-image img {
    width: 100%;
    height: auto;
    border-radius: 8px;
  }
  .project-details {
    flex: 2 1 300px;
  }
  .project-details h3 a {
    text-decoration: none;
    color: #eee;
  }
  .project-details h3 a:hover {
    text-decoration: underline;
  }
  @media (max-width: 600px) {
    .project-container {
      flex-direction: column;
    }
  }
</style>

Estos son algunos proyectos que he empezado como hobby.

<div class="project-container">
  <div class="project-image">
    <a href="https://truek.xyz" target="_blank" rel="noopener noreferrer">
      <img src="/assets/images/truek-logo.png" alt="Logo de Truek.xyz">
    </a>
  </div>
  <div class="project-details">
    <h3><a href="https://truek.xyz" target="_blank" rel="noopener noreferrer">Truek.xyz</a></h3>
    <p><em>En curso</em></p>
    <ul>
      <li>Desarrollo de una plataforma de intercambio de bienes y servicios construida con Ruby on Rails.</li>
      <li>Implementación de la lógica de intercambio, perfiles de usuario y sistemas de búsqueda avanzada.</li>
    </ul>
  </div>
</div>

<div class="project-container">
  <div class="project-image">
    <a href="https://jombo.es" target="_blank" rel="noopener noreferrer">
      <img src="/assets/images/jombo-logo.png" alt="Logo de Jombo">
    </a>
  </div>
  <div class="project-details">
    <h3><a href="https://jombo.es" target="_blank" rel="noopener noreferrer">Jombo</a></h3>
    <ul>
      <li>Evolución de la marca Jombo, actualmente centrada en el mercado finlandés.</li>
      <li>Re-arquitectura del sistema de un monolito a una API robusta para mejorar la escalabilidad y el rendimiento.</li>
    </ul>
  </div>
</div>

<div class="project-container">
  <div class="project-image">
    <a href="https://enkimail.com" target="_blank" rel="noopener noreferrer">
      <img src="/assets/images/enkimail-logo.png" alt="Logo de Enkimail">
    </a>
  </div>
  <div class="project-details">
    <h3><a href="https://enkimail.com" target="_blank" rel="noopener noreferrer">Enkimail</a></h3>
    <ul>
      <li>Desarrollo de una plataforma de email marketing de extremo a extremo centrada en la simplicidad, el rendimiento y la alta entregabilidad.</li>
      <li>Construcción de flujos de trabajo de campañas automatizadas y sistemas de gestión de listas de suscriptores a gran escala.</li>
      <li>Optimización de la infraestructura del servidor de correo y los protocolos de autenticación (SPF, DKIM, DMARC) para garantizar la máxima colocación en la bandeja de entrada.</li>
    </ul>
  </div>
</div>
