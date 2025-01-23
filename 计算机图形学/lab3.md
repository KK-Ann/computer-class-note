# lab3
## result
![[Pasted image 20241225033121.png]]
## 代码与思路
- 分成3钟材质
	- DIFFUSE，使用Phong着色模型
	- GLASS，进行折射，反射计算
	- LIGHT，直接返回光源的颜色
		- return mat_ptr->albedo;
### Phong着色
- Ambient + diffuse + specular
- 环境光 + 漫反射 + 镜面反射
- 环境光
	- 不同的表面可以有不同的环境反射系数 k（0 ≤ k≤ 1），因此，如果只考虑环境照明，则某一点的照度仅为 I= L* k
- 判断是否在阴影中，即判断从交点到光源的光线是否被遮挡
	-  scene->hit(shadow_ray, 0.001f, glm::length(light->center - rec.p), shadow_rec)
	- 若是则不进行下面的运算
	- 注意，源代码将光源也作为object，因此需判断从交点到光源的光线不是与光源相交
- 漫反射
	- 兰伯特定律
	- 强度取决于入射光的角度：光线与表面法线的夹角
	- 假设所有方向均等
	- $I_d=k_d*cos \theta *L（\theta<90)$
		- $cos \theta =$glm::max(glm::dot(rec.normal, light_dir), 0.0f);
	- $I_d=0（\theta大于90）$
- 镜面反射
	- ·Shiny surfaces have high specular coefficientα 
	- The higher α is , the narrower specular light
	-  $Is = max(k_s L_s cos^αϕ, 0.0)$
		- $cos^αϕ=$glm::pow(glm::max(glm::dot(view_dir, reflect_dir), 0.0f),α );
	- cosϕ = r 与v的夹角
- summary：$I = L_ak_a + L_dk_d max(cos(l 与 n的夹角 ),0) + L_sk_smax(cos^α(r与 v的夹角) , 0)$
- 代码
```c
if (mat_ptr->type == MatType::DIFFUSE) {
	vec3 finalColor(0.0f);

	// 计算环境光分量
	finalColor += ka * mat_ptr->albedo;

	// 计算光照贡献
	for (const auto& light : scene->lights) {
	vec3 light_dir = glm::normalize(light->center - rec.p);  // 假设光源是一个点光源
	
   // finalColor += diff * mat_ptr->albedo * light->mat_ptr->albedo;  // 漫反射光
	// 阴影检测：判断从交点到光源的光线是否被遮挡
	Ray shadow_ray(rec.p, light_dir);
	hit_record shadow_rec;
	bool in_shadow = scene->hit(shadow_ray, 0.001f, glm::length(light->center - rec.p), shadow_rec);

	// 如果没有物体遮挡光线，才计算光照
	if (!in_shadow || shadow_rec.object == light) {
		// std::cout << "没被遮挡" << std::endl;
		float diff = glm::max(glm::dot(rec.normal, light_dir), 0.0f);
		finalColor += diff * mat_ptr->albedo * light->mat_ptr->albedo;  // 漫反射光

		// 计算镜面反射分量
		vec3 view_dir = glm::normalize(ray.orig - rec.p);//观察方向向量
		vec3 reflect_dir = glm::reflect(-light_dir, rec.normal);// 反射方向
		float spec=glm::pow(glm::max(glm::dot(view_dir, reflect_dir), 0.0f), 32);
		finalColor += spec * light->mat_ptr->albedo ;  // 镜面反射光
	}            
}
	return finalColor;
}
```
- 对比不进行反射漫反射的计算
![[Pasted image 20241225042700.png|300]]
### 玻璃材质
![[Image_176590854098931_edit_177048850433652.png]]

- 对于玻璃材质需考虑折射和反射的光线的倒影
- 反射
	- 计算反射光方向reflect_dir
	- 得到reflect_ray
	- 递归调用得 reflect_color
```c
// 计算反射光线
        vec3 reflect_dir = glm::reflect(glm::normalize(ray.dir), rec.normal);
        Ray reflect_ray(rec.p, reflect_dir);
        reflect_color = trace(reflect_ray, scene, depth + 1);  // 递归调用
```
- 折射
	- 计算折射光线
		- 物理背景：斯涅尔定律（Snell's Law）
		- $n_1 \sin \theta_1 = n_2 \sin \theta_2$
		- $\sin \theta_2 = \frac{n_1}{n_2} \sin \theta_1$
		- n1和 n2的比值被称为折射比,代码中refraction_ratio = mat_ptr->refraction_ratio
		- 计算入射光线和法线之间的夹角
			- float cos_theta = glm::dot(-glm::normalize(ray.dir), rec.normal);
		- 计算折射角的正弦值
			- float sin_theta = sqrt(1.0f - cos_theta * cos_theta);
		- 判断是否发生全内反射
			- refraction_ratio * sin_theta > 1.0f;
			- 是，则refract_color = reflect_color;运用已得到的结果
			- 否则，计算折射光线的方向
				- $\cos \theta_2=$sqrt(1.0f - refraction_ratio * refraction_ratio * (1.0f - cos_theta * cos_theta))
				- $\vec{r_{\text{refract}}} = \frac{n_1}{n_2} \cdot \vec{r_{\text{incident}}} + \left( \frac{n_1}{n_2} \cdot \cos \theta_1 - \cos \theta_2 \right) \cdot \vec{n}$
				- 得到refract_ray
				- 递归得refract_color = trace(refract_ray, scene, depth + 1);
```c
// 计算折射光线
float refraction_ratio = mat_ptr->refraction_ratio;
float cos_theta = glm::dot(-glm::normalize(ray.dir), rec.normal);
float sin_theta = sqrt(1.0f - cos_theta * cos_theta);

bool is_total_internal_reflection = refraction_ratio * sin_theta > 1.0f;
if (is_total_internal_reflection) {
	// 如果发生全内反射，折射光线与反射光线相同
	refract_color = reflect_color;
}
else {
	float refract_cos = sqrt(1.0f - refraction_ratio * refraction_ratio * (1.0f - cos_theta * cos_theta));
	vec3 refract_dir = refraction_ratio * ray.dir + (refraction_ratio * cos_theta - refract_cos) * rec.normal;
	Ray refract_ray(rec.p, refract_dir);
	refract_color = trace(refract_ray, scene, depth + 1);  // 递归调用
}
```
